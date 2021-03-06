# virtio-balloon

 * https://access.redhat.com/documentation/ja-JP/Red_Hat_Enterprise_Linux/6/html/Virtualization_Getting_Started_Guide/para-virtdevices.html
 * https://rwmj.wordpress.com/2010/07/17/virtio-balloon/
 * http://d.hatena.ne.jp/kvm/20081016/1224171825

ballon inflate/deflate がどんな風に抽象化されているかを追っていく

 * struct address_space
 * struct list_head
 * virtio API
 * madvise(2)

# **balloon** の実態

```c
struct virtio_balloon {
	struct virtio_device *vdev;
	struct virtqueue *inflate_vq, *deflate_vq, *stats_vq;

	/* Where the ballooning thread waits for config to change. */
	wait_queue_head_t config_change;

	/* The thread servicing the balloon. */
	struct task_struct *thread;

	/* Waiting for host to ack the pages we released. */
	wait_queue_head_t acked;

	/* Number of balloon pages we've told the Host we're not using. */
	unsigned int num_pages;
	/*
	 * The pages we've told the Host we're not using are enqueued
	 * at vb_dev_info->pages list.
	 * Each page on this list adds VIRTIO_BALLOON_PAGES_PER_PAGE
	 * to num_pages above.
	 */
	struct balloon_dev_info *vb_dev_info;

	/* Synchronize access/update to this struct virtio_balloon elements */
	struct mutex balloon_lock;

	/* The array of pfns we tell the Host about. */
	unsigned int num_pfns;
	u32 pfns[VIRTIO_BALLOON_ARRAY_PFNS_MAX];

	/* Memory statistics */
	int need_stats_update;
	struct virtio_balloon_stat stats[VIRTIO_BALLOON_S_NR];
};
```

```
/*
 * Balloon device information descriptor.
 * This struct is used to allow the common balloon compaction interface
 * procedures to find the proper balloon device holding memory pages they'll
 * have to cope for page compaction / migration, as well as it serves the
 * balloon driver as a page book-keeper for its registered balloon devices.
 */
struct balloon_dev_info {
	void *balloon_device;		/* balloon device descriptor */
	struct address_space *mapping;	/* balloon special page->mapping */
	unsigned long isolated_pages;	/* # of isolated pages for migration */
	spinlock_t pages_lock;		/* Protection to pages list */
	struct list_head pages;		/* Pages enqueued & handled to Host */
};
```

# Guest

# ゲストカーネルで動く **vballoon** カーネルスレッド

```c
static int balloon(void *_vballoon)
{
	struct virtio_balloon *vb = _vballoon;

	set_freezable();
	while (!kthread_should_stop()) {
		s64 diff;

		try_to_freeze();
		wait_event_interruptible(vb->config_change,
					 (diff = towards_target(vb)) != 0
					 || vb->need_stats_update
					 || kthread_should_stop()
					 || freezing(current));
		if (vb->need_stats_update)
			stats_handle_request(vb);
		if (diff > 0)
			fill_balloon(vb, diff);
		else if (diff < 0)
			leak_balloon(vb, -diff);
		update_balloon_size(vb);
	}
	return 0;
}
```

wait_event_interruptible でイベントを待つ

# inflate: 風船を凸

fill_balloon から処理をはじめる

 * ゲストの RAM を未使用に指定
 * ホストでは RAM が解放されたように見える?

##### 実装 overview

ゲストで alloc_page して、ページを b_dev_info->pages に list_add する

```
fill_balloon
 -> balloon_page_enqueue
    -> struct page *page =
      alloc_page(balloon_mapping_gfp_mask() | __GFP_NOMEMALLOC | __GFP_NORETRY);
      totalram_pages--;               
    -> balloon_page_insert
      -> page->mapping = mapping;
      -> list_add(&page->lru, head);
 -> tell_host(vb, vb->inflate_vq);
   -> virtqueue_add_outbuf
   -> virtqueue_kick
```

バルーンに溜め込む page の gfp フラグは下記の通り

```c
#define GFP_HIGHUSER_MOVABLE	(__GFP_WAIT | __GFP_IO | __GFP_FS | \
				 __GFP_HARDWALL | __GFP_HIGHMEM | \
				 __GFP_MOVABLE)
#define __GFP_NOMEMALLOC ((__force gfp_t)___GFP_NOMEMALLOC) /* Don't use emergency reserves.
							 * This takes precedence over the
							 * __GFP_MEMALLOC flag if both are
							 * set
							 */
#define __GFP_NORETRY	((__force gfp_t)___GFP_NORETRY) /* See above */                 
```

GFP_HIGHUSER_MOVABLE は 以下の論理和

```
#define __GFP_WAIT	((__force gfp_t)___GFP_WAIT)	/* Can wait and reschedule? */
#define __GFP_IO	((__force gfp_t)___GFP_IO)	/* Can start physical IO? */
#define __GFP_FS	((__force gfp_t)___GFP_FS)	/* Can call down to low-level FS? */
#define __GFP_HARDWALL   ((__force gfp_t)___GFP_HARDWALL) /* Enforce hardwall cpuset memory allocs */
#define __GFP_HIGHMEM	((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_MOVABLE	((__force gfp_t)___GFP_MOVABLE)  /* Page is movable */
```

ページをどのように扱うか?

 * b_dev_info->pages に繋いだページは、qemu から madvise(2) + MADV_DONTNEED されて zap され、ホストから再利用可能になる
 * b_dev_info->pages に繋いだページは、qemu から madvise(2) + MADV_WILLNEED されて ゲスト用に予約される???

# deflate: 風船を凹

leak_balloon から処理をはじめる

 * ゲストで 未使用に指定していた RAM を減らす

##### 実装 overview

```
leak_balloon
-> balloon_page_dequeue
  -> balloon_page_delete
    * b_dev_info->pages (LRU) から page を list_del する
    -> page->mapping = NULL;
    -> list_del(&page->lru);
 -> balloon_page_free
  -> release_pages_by_pfn
    * ページの「解放」
     -> put_page
     -> __free_pages
     -> totalram_pages++
```

# stats

ホストからゲストのメモリの統計をとれる

 * https://github.com/01org/KVMGT-qemu/blob/master/docs/virtio-balloon-stats.txt
 * virDomainMemoryStats

##### 実装

ゲスト内で si_meminfo() を使ってメモリの統計をとっている ( /proc/meminfo と同等のデータを取れる )

```c
static void update_balloon_stats(struct virtio_balloon *vb)
{
	unsigned long events[NR_VM_EVENT_ITEMS];
	struct sysinfo i;
	int idx = 0;

	all_vm_events(events);
	si_meminfo(&i);

	update_stat(vb, idx++, VIRTIO_BALLOON_S_SWAP_IN,
				pages_to_bytes(events[PSWPIN]));
	update_stat(vb, idx++, VIRTIO_BALLOON_S_SWAP_OUT,
				pages_to_bytes(events[PSWPOUT]));
	update_stat(vb, idx++, VIRTIO_BALLOON_S_MAJFLT, events[PGMAJFAULT]);
	update_stat(vb, idx++, VIRTIO_BALLOON_S_MINFLT, events[PGFAULT]);
	update_stat(vb, idx++, VIRTIO_BALLOON_S_MEMFREE,
				pages_to_bytes(i.freeram));
	update_stat(vb, idx++, VIRTIO_BALLOON_S_MEMTOT,
				pages_to_bytes(i.totalram));
}
```

----

# qemu

> a. There is no host balloon driver, at least no special host kernel code for virtio-balloon. virtio-balloon is implemented in QEMU userspace, see hw/virtio/virtio-balloon.c.
>
> http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html

 * ホストカーネルでのドライバ実装は無くて QEMU プロセスだけで実装されている ( madvise(2) の利用 )
 * hw/virtio/virtio-balloon を読むとよい

## balloon で確保したページをどう扱うのか?

##### ballon の inflate

 * madvise(2) もしくは posix_madsive(2) で MADV_DONTNEED して zap する
 * ballon のページはゲスト(プロセス)が確保しているページなので zap すると、結果、空きメモリが増える = ホストに返却されるということ

##### ballon の deflate

 * madvise(2) もしくは posix_madsive(2) で MADV_WILLNEED
 * mapping->a_ops->readpage, mapping->a_ops->readpages が無いようだけど?

# 実装

```c
static void balloon_page(void *addr, int deflate)
{
#if defined(__linux__)
    if (!qemu_balloon_is_inhibited() && (!kvm_enabled() ||
                                         kvm_has_sync_mmu())) {
        qemu_madvise(addr, BALLOON_PAGE_SIZE,
                deflate ? QEMU_MADV_WILLNEED : QEMU_MADV_DONTNEED);
    }
#endif
}
```

# virtio-device の実装

virtqueue はホストからゲストへコマンドを送り込むためのインタフェースとして使われている

 * inflate
 * deflate
 * stats

```c
static void virtio_balloon_device_realize(DeviceState *dev, Error **errp)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(dev);
    VirtIOBalloon *s = VIRTIO_BALLOON(dev);
    int ret;

    // ?
    virtio_init(vdev, "virtio-balloon", VIRTIO_ID_BALLOON,
                sizeof(struct virtio_balloon_config));

    ret = qemu_add_balloon_handler(virtio_balloon_to_target,
                                   virtio_balloon_stat, s);

    if (ret < 0) {
        error_setg(errp, "Only one balloon device is supported");
        virtio_cleanup(vdev);
        return;
    }

    // inflate
    s->ivq = virtio_add_queue(vdev, 128, virtio_balloon_handle_output);
    // deflate
    s->dvq = virtio_add_queue(vdev, 128, virtio_balloon_handle_output);
    // stats
    s->svq = virtio_add_queue(vdev, 128, virtio_balloon_receive_stats);

    reset_stats(s);
}
```

```c
static void virtio_balloon_handle_output(VirtIODevice *vdev, VirtQueue *vq)
{
    VirtIOBalloon *s = VIRTIO_BALLOON(vdev);
    VirtQueueElement *elem;
    MemoryRegionSection section;

    for (;;) {
        size_t offset = 0;
        uint32_t pfn;
        elem = virtqueue_pop(vq, sizeof(VirtQueueElement));
        if (!elem) {
            return;
        }

        while (iov_to_buf(elem->out_sg, elem->out_num, offset, &pfn, 4) == 4) {
            ram_addr_t pa;
            ram_addr_t addr;
            int p = virtio_ldl_p(vdev, &pfn);

            pa = (ram_addr_t) p << VIRTIO_BALLOON_PFN_SHIFT;
            offset += 4;

            /* FIXME: remove get_system_memory(), but how? */
            section = memory_region_find(get_system_memory(), pa, 1);
            if (!int128_nz(section.size) || !memory_region_is_ram(section.mr))
                continue;

            trace_virtio_balloon_handle_output(memory_region_name(section.mr),
                                               pa);
            /* Using memory_region_get_ram_ptr is bending the rules a bit, but
               should be OK because we only want a single page.  */
            addr = section.offset_within_region;

            // ここで madvise して page を操作する
            balloon_page(memory_region_get_ram_ptr(section.mr) + addr,
                         !!(vq == s->dvq));
            memory_region_unref(section.mr);
        }

        virtqueue_push(vq, elem, offset);
        virtio_notify(vdev, vq);
        g_free(elem);
    }
}
```

# kernel の CONFIG

##### CONFIG_BALLOON_COMPACTION

 * http://cateee.net/lkddb/web-lkddb/BALLOON_COMPACTION.html
 * ballon によってメモリのフラグメンテーションが起きて THP の性能劣化を招くのを防ぐた目に compaction 可能にする?