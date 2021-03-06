# xfs 調べもの

## API

 * work_struct
   * INIT_WORK
   * cancel_work_sync
     * プロセスコンテキスト
     * http://www.ibm.com/developerworks/jp/linux/library/l-tasklets/
 * delayed_works
   * INIT_DELAYED_WORK
   * cancel_delayed_work_sync
 * queue_work

xfs の内部は辛そうなのであんま触れない

 * xfs_flush_worker
   * `/* background inode flush */`
 * xfs_sync_worker
   * delayed_works 
   * `/* background sync work */`
 * xfs_reclaim_inodes
   * delayed_works 
   * `/* background inode reclaim */`

## queue_work    

```   
/*
 * Flush delayed allocate data, attempting to free up reserved space
 * from existing allocations.  At this point a new allocation attempt
 * has failed with ENOSPC and we are in the process of scratching our
 * heads, looking about for more room.
 *
 * Queue a new data flush if there isn't one already in progress and
 * wait for completion of the flush. This means that we only ever have one
 * inode flush in progress no matter how many ENOSPC events are occurring and
 * so will prevent the system from bogging down due to every concurrent
 * ENOSPC event scanning all the active inodes in the system for writeback.
 */
void
xfs_flush_inodes(
	struct xfs_inode	*ip)
{
	struct xfs_mount	*mp = ip->i_mount;

    // ここで m_flush_work を呼び出す
	queue_work(xfs_syncd_wq, &mp->m_flush_work);
	flush_work_sync(&mp->m_flush_work);
}
```   

## xfs_flush_worker

 * カーネルスレッドのエントリポイントは以下のコード

```c
STATIC void
xfs_flush_worker(
	struct work_struct *work)
{
	struct xfs_mount *mp = container_of(work,
					struct xfs_mount, m_flush_work);

	xfs_sync_data(mp, SYNC_TRYLOCK);
	xfs_sync_data(mp, SYNC_TRYLOCK | SYNC_WAIT);
}
```

xfs_flush_worker がポインタで渡されてる箇所

```c
int
xfs_syncd_init(
	struct xfs_mount	*mp)
{
	INIT_WORK(&mp->m_flush_work, xfs_flush_worker);
```

**background inode flush** という役割

```c
	struct work_struct	m_flush_work;	/* background inode flush */
```    

## xfs_sync_data

```c
/*
 * Write out pagecache data for the whole filesystem.
 */
STATIC int
xfs_sync_data(
	struct xfs_mount	*mp,
	int			flags)
{
	int			error;

	ASSERT((flags & ~(SYNC_TRYLOCK|SYNC_WAIT)) == 0);

	error = xfs_inode_ag_iterator(mp, xfs_sync_inode_data, flags);
	if (error)
		return XFS_ERROR(error);

	xfs_log_force(mp, (flags & SYNC_WAIT) ? XFS_LOG_SYNC : 0);
	return 0;
}
```

```c
int
xfs_inode_ag_iterator(
	struct xfs_mount	*mp,
	int			(*execute)(struct xfs_inode *ip,
					   struct xfs_perag *pag, int flags),
	int			flags)
{
	struct xfs_perag	*pag;
	int			error = 0;
	int			last_error = 0;
	xfs_agnumber_t		ag;

	ag = 0;
	while ((pag = xfs_perag_get(mp, ag))) {
		ag = pag->pag_agno + 1;
        // execute がコールバック
        // radix_tree 
		error = xfs_inode_ag_walk(mp, pag, execute, flags);
		xfs_perag_put(pag);
		if (error) {
			last_error = error;
			if (error == EFSCORRUPTED)
				break;
		}
	}
	return XFS_ERROR(last_error);
}
```

```c
STATIC int
xfs_inode_ag_walk(
	struct xfs_mount	*mp,
	struct xfs_perag	*pag,
	int			(*execute)(struct xfs_inode *ip,
					   struct xfs_perag *pag, int flags),
	int			flags)
{
	uint32_t		first_index;
	int			last_error = 0;
	int			skipped;
	int			done;
	int			nr_found;

restart:
	done = 0;
	skipped = 0;
	first_index = 0;
	nr_found = 0;
	do {
		struct xfs_inode *batch[XFS_LOOKUP_BATCH];
		int		error = 0;
		int		i;

		rcu_read_lock();
		nr_found = radix_tree_gang_lookup(&pag->pag_ici_root,
					(void **)batch, first_index,
					XFS_LOOKUP_BATCH);
		if (!nr_found) {
			rcu_read_unlock();
			break;
		}

		/*
		 * Grab the inodes before we drop the lock. if we found
		 * nothing, nr == 0 and the loop will be skipped.
		 */
		for (i = 0; i < nr_found; i++) {
			struct xfs_inode *ip = batch[i];

			if (done || xfs_inode_ag_walk_grab(ip))
				batch[i] = NULL;

			/*
			 * Update the index for the next lookup. Catch
			 * overflows into the next AG range which can occur if
			 * we have inodes in the last block of the AG and we
			 * are currently pointing to the last inode.
			 *
			 * Because we may see inodes that are from the wrong AG
			 * due to RCU freeing and reallocation, only update the
			 * index if it lies in this AG. It was a race that lead
			 * us to see this inode, so another lookup from the
			 * same index will not find it again.
			 */
			if (XFS_INO_TO_AGNO(mp, ip->i_ino) != pag->pag_agno)
				continue;
			first_index = XFS_INO_TO_AGINO(mp, ip->i_ino + 1);
			if (first_index < XFS_INO_TO_AGINO(mp, ip->i_ino))
				done = 1;
		}

		/* unlock now we've grabbed the inodes. */
		rcu_read_unlock();

		for (i = 0; i < nr_found; i++) {
			if (!batch[i])
				continue;

            // ここでコールバック呼び出し                
			error = execute(batch[i], pag, flags);
			IRELE(batch[i]);
			if (error == EAGAIN) {
				skipped++;
				continue;
			}
			if (error && last_error != EFSCORRUPTED)
				last_error = error;
		}

		/* bail out if the filesystem is corrupted.  */
		if (error == EFSCORRUPTED)
			break;

		cond_resched();

	} while (nr_found && !done);

	if (skipped) {
		delay(1);
		goto restart;
	}
	return last_error;
}
```

xfs_sync_data のコールバック

```
STATIC int
xfs_sync_inode_data(
	struct xfs_inode	*ip,
	struct xfs_perag	*pag,
	int			flags)
{
	struct inode		*inode = VFS_I(ip);
	struct address_space *mapping = inode->i_mapping;
	int			error = 0;

	if (!mapping_tagged(mapping, PAGECACHE_TAG_DIRTY))
		return 0;

	if (!xfs_ilock_nowait(ip, XFS_IOLOCK_SHARED)) {
		if (flags & SYNC_TRYLOCK)
			return 0;
		xfs_ilock(ip, XFS_IOLOCK_SHARED);
	}

	error = xfs_flush_pages(ip, 0, -1, (flags & SYNC_WAIT) ?
				0 : XBF_ASYNC, FI_NONE);
	xfs_iunlock(ip, XFS_IOLOCK_SHARED);
	return error;}
```

```
int
xfs_flush_pages(
	xfs_inode_t	*ip,
	xfs_off_t	first,
	xfs_off_t	last,
	uint64_t	flags,
	int		fiopt)
{
	struct address_space *mapping = VFS_I(ip)->i_mapping;
	int		ret = 0;
	int		ret2;

	xfs_iflags_clear(ip, XFS_ITRUNCATED);
	ret = -filemap_fdatawrite_range(mapping, first,
				last == -1 ? LLONG_MAX : last);
	if (flags & XBF_ASYNC)
		return ret;
	ret2 = xfs_wait_on_pages(ip, first, last);
	if (!ret)
		ret = ret2;
	return ret;
}
```

これ以下は address_space の実装に依る

```c
int
xfs_wait_on_pages(
	xfs_inode_t	*ip,
	xfs_off_t	first,
	xfs_off_t	last)
{
	struct address_space *mapping = VFS_I(ip)->i_mapping;

	if (mapping_tagged(mapping, PAGECACHE_TAG_WRITEBACK)) {
		return -filemap_fdatawait_range(mapping, first,
					last == -1 ? ip->i_size - 1 : last);
	}
	return 0;
}
```

## write から flush_work_sync へのパス

```
[hiroya@*******]~% sudo cat /proc/$( pgrep find )/stack
[<ffffffff81083a6a>] flush_work_sync+0x5a/0x70
[<ffffffffa01b9dff>] xfs_flush_inodes+0x2f/0x40 [xfs]
[<ffffffffa01b381a>] xfs_iomap_write_delay+0x13a/0x2e0 [xfs]
[<ffffffffa01a5ee3>] __xfs_get_blocks+0x2d3/0x460 [xfs]
[<ffffffffa01a60a1>] xfs_get_blocks+0x11/0x20 [xfs]
[<ffffffff811a3c0c>] __block_write_begin+0x1fc/0x570
[<ffffffff811a4206>] block_write_begin+0x56/0x90
[<ffffffffa01a5161>] xfs_vm_write_begin+0x41/0x70 [xfs]
[<ffffffff8110d9da>] generic_perform_write+0xca/0x210
[<ffffffff8110db84>] generic_file_buffered_write+0x64/0xa0
[<ffffffffa01ac571>] xfs_file_buffered_aio_write+0xf1/0x1a0 [xfs]
[<ffffffffa01aca17>] xfs_file_aio_write+0x1b7/0x290 [xfs]
[<ffffffff8116e752>] do_sync_write+0xe2/0x120
[<ffffffff8116ed13>] vfs_write+0xf3/0x1f0
[<ffffffff8116ef11>] sys_write+0x51/0x90
[<ffffffff81527a7d>] system_call_fastpath+0x18/0x1d
[<ffffffffffffffff>] 0xffffffffffffffff

```
