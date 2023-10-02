# Page

页帧page是物理内存管理的基本单位，`struct page`记录了任意时刻page的所有状态，因此每一个物理页帧都需一个对应的`struct page`结构体记录状态，对于内存多计算机系统需要的`struct page`本身就需要大量内存进行存储，因此该结构体中每增加一个变量带来的代价会很大，需要仔细控制该结构体的size。内核广泛使用了union，同一时刻不会同时使用的变量作为一个union减少内存开销。

## struct page

`struct page`的声明如下：

- `flags`: 标志位
- `_count`: 使用计数，关系到内存释放。
- `_mapcount` & `inuse`: `_mapcount`是页表项的映射个数，当page用于slab分配器管理时使用`inuse`表示包含的对象个数。
- `priave`: page用作pagecahe时`private`存放`struct buffer_head`;page内容被换出时，private存放`swap_entry_t`;page位于伙伴系统时，存放`order`
- `mapping`: 如果low bit没有set，指向inode的`address_space`，如果有low bit，指向`aono_vma`对象，此时clear低位bit后就可以访问到有效指针。指针都是字节对齐的，低位bit正常情况为0，可以在低位bit中添加信息。
- `slab`: 指向所属的slab分配器`kmem_cache`结构体。
- `first_page`: 用于复合页中tail page指向head page。
- `virtual`: 高端内存页的虚拟地址。

```c
struct page {
    unsigned long flags;  /* Atomic flags, some possibly
                     * updated asynchronously */
    atomic_t _count;  /* Usage count, see below. */
    union {
        atomic_t _mapcount; /* Count of ptes mapped in mms,
                     * to show when page is mapped
                     * & limit reverse map searches.
                     */
        unsigned int inuse; /* SLUB: Nr of objects */
    };
    union {
        struct {
        unsigned long private;  /* Mapping-private opaque data:
                          * usually used for buffer_heads
                         * if PagePrivate set; used for
                         * swp_entry_t if PageSwapCache;
                         * indicates order in the buddy
                         * system if PG_buddy is set.
                         */
        struct address_space *mapping; /* If low bit clear, points to
                         * inode address_space, or NULL.
                         * If page mapped as anonymous
                         * memory, low bit is set, and
                         * it points to anon_vma object:
                         * see PAGE_MAPPING_ANON below.
                         */
        };
#if NR_CPUS >= CONFIG_SPLIT_PTLOCK_CPUS
        spinlock_t ptl;
#endif
        struct kmem_cache *slab; /* SLUB: Pointer to slab */
        struct page *first_page; /* Compound tail pages */
    };
    union {
        pgoff_t index;  /* Our offset within mapping. */
        void *freelist;  /* SLUB: freelist req. slab lock */
    };
    struct list_head lru;  /* Pageout list, eg. active_list
                     * protected by zone->lru_lock !
                     */
    /*
     * On machines where all RAM is mapped into kernel address space,
     * we can simply calculate the virtual address. On machines with
     * highmem some memory is mapped into kernel virtual memory
     * dynamically, so we need a place to store that address.
     * Note that this field could be 16 bits on x86 ... ;)
     *
     * Architectures with slow multiplication can define
     * WANT_PAGE_VIRTUAL in asm/page.h
     */
#if defined(WANT_PAGE_VIRTUAL)
    void *virtual;   /* Kernel virtual address (NULL if
                       not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */
#ifdef CONFIG_CGROUP_MEM_RES_CTLR
    unsigned long page_cgroup;
#endif
};
```

swap、地址映射、复合页、slab分配器等机制都会操作page结构体。

## PG_FLAG

- PG_locked: 锁定，避免竞态。
- PG_error: IO error
- PG_reference & PG_active: 和page swap相关。
- PG_uptodate: 数据已从块设备读取并且未出错。
- PG_dirty: 数据未同步到磁盘。
- PG_lru: 页面回收和切换相关，表示在active_list or inactive_list中，如果PG_active同时设置表明在active_list中。
- PG_slab: 用于slab分配器。
- PG_private: private变量不为空时使用。
- PG_writeback: 处于IO写回的过程中。
- PG_compound: 复合页。
- PG_swapcache: 处于swap cache。
- PG_reclaim: 内核决定回收该page时需要标记可回收。
- PG_buddy: 位于伙伴系统中，空闲状态。

```c
#define PG_locked    0 /* Page is locked. Don't touch. */
#define PG_error   1
#define PG_referenced   2
#define PG_uptodate   3

#define PG_dirty    4
#define PG_lru    5
#define PG_active   6
#define PG_slab    7 /* slab debug (Suparna wants this) */

#define PG_owner_priv_1   8 /* Owner use. If pagecache, fs may use*/
#define PG_arch_1   9
#define PG_reserved  10
#define PG_private  11 /* If pagecache, has fs-private data */

#define PG_writeback  12 /* Page is under writeback */
#define PG_compound  14 /* Part of a compound page */
#define PG_swapcache  15 /* Swap page: swp_entry_t in private */

#define PG_mappedtodisk  16 /* Has blocks allocated on-disk */
#define PG_reclaim  17 /* To be reclaimed asap */
#define PG_buddy  19 /* Page is free, on buddy lists */

/* PG_readahead is only used for file reads; PG_reclaim is only for writes */
#define PG_readahead  PG_reclaim /* Reminder to do async read-ahead */

/* PG_owner_priv_1 users should have descriptive aliases */
#define PG_checked  PG_owner_priv_1 /* Used by some filesystems */
#define PG_pinned  PG_owner_priv_1 /* Xen pinned pagetable */

```
