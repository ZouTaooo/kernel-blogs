# 分配释放的是什么

page的分配和释放实际上是对物理页面的状态进行管理，页面的分配实际上就是找到一个处于free状态的物理页面，建立其与虚拟空间page的映射，页面的释放就是取消这种映射关系，让其回归free状态，能够重新被分配。

# 交换的是什么

交换指的是物理页面与磁盘文件交换区的页面交换，但是需要注意的是，并不是真的在物理层面上做了交换，交换的东西只有page的内容。

# 为什么要交换

交换发生在物理内存页面不足时，但是进程的数据存放在内存中又无法丢弃，如果别的进程产生缺页异常，尝试分配页面就会失败。交换作为一种时间换空间的技术，作用就是将一些不活跃的页面暂放在磁盘的设备中，释放出一定的物理内存页面，换取更大的能够使用的内存存储空间。

# 交换存在的问题

1. 换入换出存在开销：交换到交换区的页面，需要进行换出，如果再次访问该页面需要重新分配物理页面，并将内容拷贝回来，开销较大，因此实时性要求较高的系统一般不会开启页面交换
2. 抖动问题：刚被换出的物理页面又可能又被访问，然后再次被换出，极端情况下系统的性能下降很大。

# 页面交换设备(交换区)管理

在kernel中使用swap_info_struct这个结构体来描述一个交换区对象。其中，flags中包含一些标识位，prio表示交换区的优先级，swap_map存放了每一个交换区上page的引用计数，为0表示该page是空闲的，可以换入。cluster相关的是针对SSD的一种优化，将相邻的多个page看作一个cluster，再用链表串起来，降低锁的竞争。

```c
struct swap_info_struct {
	unsigned long	flags;		
	signed short	prio;		/* swap priority of this type */
	struct plist_node list;		/* entry in swap_active_head */
	signed char	type;		/* strange name for an index */
	unsigned int	max;		/* extent of the swap_map */
	unsigned char *swap_map;	/* vmalloc'ed array of usage counts */
	struct swap_cluster_info *cluster_info; /* cluster info. Only for SSD */
	struct swap_cluster_list free_clusters; /* free clusters list */
	unsigned int lowest_bit;	/* index of first free in swap_map */
	unsigned int highest_bit;	/* index of last free in swap_map */
	unsigned int pages;		/* total of usable pages of swap */
	unsigned int inuse_pages;	/* number of those currently in use */
	unsigned int cluster_next;	/* likely index for next allocation */
	unsigned int cluster_nr;	/* countdown to next cluster search */
	struct percpu_cluster __percpu *percpu_cluster; /* per cpu's swap location */
	struct swap_extent *curr_swap_extent;
	struct swap_extent first_swap_extent;
	struct block_device *bdev;	/* swap device or bdev of swap file */
	struct file *swap_file;		/* seldom referenced */
	unsigned int old_block_size;	/* seldom referenced */
#ifdef CONFIG_FRONTSWAP
	unsigned long *frontswap_map;	/* frontswap in-use, one bit per page */
	atomic_t frontswap_pages;	/* frontswap pages in-use counter */
#endif
	spinlock_t lock;		/*
					 * protect map scan related fields like
					 * swap_map, lowest_bit, highest_bit,
					 * inuse_pages, cluster_next,
					 * cluster_nr, lowest_alloc,
					 * highest_alloc, free/discard cluster
					 * list. other fields are only changed
					 * at swapon/swapoff, so are protected
					 * by swap_lock. changing flags need
					 * hold this lock and swap_lock. If
					 * both locks need hold, hold swap_lock
					 * first.
					 */
	spinlock_t cont_lock;		/*
					 * protect swap count continuation page
					 * list.
					 */
	struct work_struct discard_work; /* discard worker */
	struct swap_cluster_list discard_clusters; /* discard clusters list */
	struct plist_node avail_lists[0]; /*
					   * entries in swap_avail_heads, one
					   * entry per node.
					   * Must be last as the number of the
					   * array is nr_node_ids, which is not
					   * a fixed value so have to allocate
					   * dynamically.
					   * And it has to be an array so that
					   * plist_for_each_* can work.
					   */
};
```
