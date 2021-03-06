## ___/proc/\<pid\>/smaps___

## KernelPageSize, KernelPageSize

 * vm_area_struct に割り当てた ページサイズなのかな?
 * だいたいは PTE = ページテーブルエントリのサイズと一緒とのこと

```
		   "KernelPageSize: %8lu kB\n"
		   "MMUPageSize:    %8lu kB\n",
// ...
		   vma_kernel_pagesize(vma) >> 10,
		   vma_mmu_pagesize(vma) >> 10);
```

```c
/*
 * Return the size of the pages allocated when backing a VMA. In the majority
 * cases this will be same size as used by the page table entries.
 */
unsigned long vma_kernel_pagesize(struct vm_area_struct *vma)
{
	struct hstate *hstate;

    /* VM_HUGETLB か否か */
	if (!is_vm_hugetlb_page(vma))
		return PAGE_SIZE;

	hstate = hstate_vma(vma);

	return 1UL << (hstate->order + PAGE_SHIFT);
}
EXPORT_SYMBOL_GPL(vma_kernel_pagesize);
```

vm_area_struct + MMU 用に割り当てたページ

```c
/*
 * Return the page size being used by the MMU to back a VMA. In the majority
 * of cases, the page size used by the kernel matches the MMU size. On
 * architectures where it differs, an architecture-specific version of this
 * function is required.
 */
#ifndef vma_mmu_pagesize
unsigned long vma_mmu_pagesize(struct vm_area_struct *vma)
{
	return vma_kernel_pagesize(vma);
}
#endif
```

## fs/proc/task_mmu.c

### Proportional Set Size(PSS)

PSS x share しているプロセス数 = プロセス群のRSS

```
/*
 * Proportional Set Size(PSS): my share of RSS.
 *
 * PSS of a process is the count of pages it has in memory, where each
 * page is divided by the number of processes sharing it.  So if a
 * process has 1000 pages all to itself, and 1000 shared with one other
 * process, its PSS will be 1500.
```

/proc/<pid>/smaps のコード

 * PTEを全部走査して、フラグに応じて利用種別のサイズを出す

```c
	seq_printf(m,
		   "Size:           %8lu kB\n"
		   "Rss:            %8lu kB\n" 
		   "Pss:            %8lu kB\n"   // Proportional Set Size
		   "Shared_Clean:   %8lu kB\n"
		   "Shared_Dirty:   %8lu kB\n"
		   "Private_Clean:  %8lu kB\n"
		   "Private_Dirty:  %8lu kB\n"
		   "Referenced:     %8lu kB\n"
		   "Anonymous:      %8lu kB\n"
		   "AnonHugePages:  %8lu kB\n"
		   "Swap:           %8lu kB\n"
		   "KernelPageSize: %8lu kB\n"
		   "MMUPageSize:    %8lu kB\n",
		   (vma->vm_end - vma->vm_start) >> 10,
		   mss.resident >> 10,
		   (unsigned long)(mss.pss >> (10 + PSS_SHIFT)),
		   mss.shared_clean  >> 10,
		   mss.shared_dirty  >> 10,
		   mss.private_clean >> 10,
		   mss.private_dirty >> 10,
		   mss.referenced >> 10,
		   mss.anonymous >> 10,
		   mss.anonymous_thp >> 10,
		   mss.swap >> 10,
		   vma_kernel_pagesize(vma) >> 10,
		   vma_mmu_pagesize(vma) >> 10);
```

 * pte_present
   * pte_flags(a) & (_PAGE_PRESENT | _PAGE_PROTNONE);
   * PTE は 仮想メモリ => 物理メモリのアドレスを保持する
   * `PTE contains a translation */` と説明
 * pte_file
   * pte_flags(pte) & _PAGE_FILE;
   * `/* - set: nonlinear file mapping, saved PTE; unset:swap */`
   * ?
 * pte_none
   * !pte.pte;
   * ページフレームが割り当てられていない。物理アドレスを保持していない
 * is_swap_pte = !pte_none(pte) && !pte_present(pte) && !pte_file(pte);
 * pte_dirty
   * pte_flags(pte) & (_PAGE_DIRTY | _PAGE_SOFTDIRTY);
   * PageDirty
     * ?
 * pte_young
   * pte_flags(pte) & _PAGE_ACCESSED;
   * 参照ビットが立っている
   * PageReferenced
     * ?
 * PageAnon
   * ((unsigned long)page->mapping & PAGE_MAPPING_ANON) != 0;

``` c
#define _PAGE_BIT_PRESENT	0	/* is present */
#define _PAGE_BIT_RW		1	/* writeable */
#define _PAGE_BIT_USER		2	/* userspace addressable */
#define _PAGE_BIT_PWT		3	/* page write through */
#define _PAGE_BIT_PCD		4	/* page cache disabled */
#define _PAGE_BIT_ACCESSED	5	/* was accessed (raised by CPU) */
#define _PAGE_BIT_DIRTY		6	/* was written to (raised by CPU) */
#define _PAGE_BIT_PSE		7	/* 4 MB (or 2MB) page */
#define _PAGE_BIT_PAT		7	/* on 4KB pages */
#define _PAGE_BIT_GLOBAL	8	/* Global TLB entry PPro+ */
#define _PAGE_BIT_UNUSED1	9	/* available for programmer */
#define _PAGE_BIT_IOMAP		10	/* flag used to indicate IO mapping */
#define _PAGE_BIT_HIDDEN	11	/* hidden by kmemcheck */
#define _PAGE_BIT_PAT_LARGE	12	/* On 2MB or 1GB pages */
#define _PAGE_BIT_SPECIAL	_PAGE_BIT_UNUSED1
#define _PAGE_BIT_CPA_TEST	_PAGE_BIT_UNUSED1
#define _PAGE_BIT_SOFTDIRTY	_PAGE_BIT_HIDDEN
#define _PAGE_BIT_SPLITTING	_PAGE_BIT_UNUSED1 /* only valid on a PSE pmd */
#define _PAGE_BIT_NX           63       /* No execute: only valid after cpuid check */
```

## PTEの走査

![oreilly-device-drivers-linux-page-tables-ldr2_1302](https://f.cloud.github.com/assets/172456/2341729/309c90fe-a4e2-11e3-9083-d42e46cfa687.gif)

pte_t からページの利用種別に統計を取る計算

 * ptent_size が複数の項目で加算されてるのに注意だぞう
 * page->_mapcount > 1 でプロセス間で共有されているかどうかを判定できる

```c
static void smaps_pte_entry(pte_t ptent, unsigned long addr,
		unsigned long ptent_size, struct mm_walk *walk)
{
	struct mem_size_stats *mss = walk->private;
	struct vm_area_struct *vma = mss->vma;
	struct page *page;
	int mapcount;

    // ページテーブルの内容は swap されている
	if (is_swap_pte(ptent)) {
		mss->swap += ptent_size;
		return;
	}

    // ページテーブルエントリが無いので加算されない
	if (!pte_present(ptent))
		return;

	page = vm_normal_page(vma, addr, ptent);
	if (!page)
		return;

    // 無名ページ        
	if (PageAnon(page))
		mss->anonymous += ptent_size;

	mss->resident += ptent_size;

	/* Accumulate the size in pages that have been accessed. */
	if (pte_young(ptent) || PageReferenced(page))
		mss->referenced += ptent_size;

    // atomic_read(&(page)->_mapcount) + 1;        
	mapcount = page_mapcount(page);

	if (mapcount >= 2) {
        // mapcount が 2 == 複数プロセスで shared なページ
		if (pte_dirty(ptent) || PageDirty(page))
			mss->shared_dirty += ptent_size;
		else
			mss->shared_clean += ptent_size;

        // 1プロセス分に換算して足し算
		mss->pss += (ptent_size << PSS_SHIFT) / mapcount;
	} else {
        // mapcount が 1 == private なページ
		if (pte_dirty(ptent) || PageDirty(page))
			mss->private_dirty += ptent_size;
		else
			mss->private_clean += ptent_size;
		mss->pss += (ptent_size << PSS_SHIFT);
	}
}
```

smaps_pte_entry は walk_page_range の smaps_pte_range コールバック内で使われている

```c
static int show_smap(struct seq_file *m, void *v)

// ...

	struct mm_walk smaps_walk = {
		.pmd_entry = smaps_pte_range, // コールバック
		.mm = vma->vm_mm,
		.private = &mss,
	};

	memset(&mss, 0, sizeof mss);
	mss.vma = vma;
	/* mmap_sem is held in m_start */
	if (vma->vm_mm && !is_vm_hugetlb_page(vma))
		walk_page_range(vma->vm_start, vma->vm_end, &smaps_walk);
```

 * walk_page_range -> walk_pud_range -> walk_pmd_range -> smaps_pte_range
   * PGD -> PUD -> PMD -> PTE,PTE,..., のディレクトリ/テーブル走査
 * smaps_pte_range は特定の PMD以下の PTE を順番に走査する

```c
static int smaps_pte_range(pmd_t *pmd, unsigned long addr, unsigned long end,
			   struct mm_walk *walk)
{
	struct mem_size_stats *mss = walk->private;
	struct vm_area_struct *vma = mss->vma;
	pte_t *pte;
	spinlock_t *ptl;

	spin_lock(&walk->mm->page_table_lock);
    // Page Middle Directory が HugePage (4MB or 2MB) かどうか
	if (pmd_trans_huge(*pmd)) {
        // ? 
		if (pmd_trans_splitting(*pmd)) {
			spin_unlock(&walk->mm->page_table_lock);
			wait_split_huge_page(vma->anon_vma, pmd);
		} else {
			smaps_pte_entry(*(pte_t *)pmd, addr,
					HPAGE_PMD_SIZE, walk);
			spin_unlock(&walk->mm->page_table_lock);

            // "AnonHugePages:  %8lu kB\n"
			mss->anonymous_thp += HPAGE_PMD_SIZE;
			return 0;
		}
	} else {
		spin_unlock(&walk->mm->page_table_lock);
	}

	if (pmd_trans_unstable(pmd))
		return 0;
	/*
	 * The mmap_sem held all the way back in m_start() is what
	 * keeps khugepaged out of here and from collapsing things
	 * in here.
	 */
    // PTEテーブルを順番に走査
    //
    // +---+
    // |---|
    // |---|
    // |---|\_ PAGE_SIZE のオフセット
    // |---|
    // +---+
    //
	pte = pte_offset_map_lock(vma->vm_mm, pmd, addr, &ptl);
    // PAGE_SIZE ごとにアドレスをインクリメント
	for (; addr != end; pte++, addr += PAGE_SIZE)
        // PTEがどんな用途で使われているか計算
		smaps_pte_entry(*pte, addr, PAGE_SIZE, walk);
	pte_unmap_unlock(pte - 1, ptl);
	cond_resched();
	return 0;
}
```