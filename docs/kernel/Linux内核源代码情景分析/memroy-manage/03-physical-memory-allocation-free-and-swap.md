# 内存的使用与周转

## 基础概念

### 分配释放的是什么

page的分配和释放实际上是对物理页面的状态进行管理，页面的分配实际上就是找到一个处于free状态的物理页面，建立其与虚拟空间page的映射，页面的释放就是取消这种映射关系，最终让其回归free状态，能够重新被分配。

### 交换的是什么

交换指的是物理页面与磁盘文件交换区的页面交换，但是需要注意的是，并不是真的在物理层面上做了交换，交换的东西只有page的内容。

### 为什么要交换

交换发生在物理内存页面不足时，但是进程的数据存放在内存中又无法丢弃，如果别的进程产生缺页异常，尝试分配页面就会失败。交换作为一种时间换空间的技术，作用就是将一些不活跃的页面暂放在磁盘的设备中，释放出一定的物理内存页面，换取更大的能够使用的内存存储空间。

### 交换存在的问题

1. 换入换出存在开销：交换到交换区的页面，需要进行换出，如果再次访问该页面需要重新分配物理页面，并将内容拷贝回来，开销较大，因此实时性要求较高的系统一般不会开启页面交换
2. 抖动问题：刚被换出的物理页面又可能又被访问，然后再次被换出，极端情况下系统的性能下降很大。

## 页面交换设备(交换区)管理

在kernel中使用`swap_info_struct`这个结构体来描述一个交换区对象。其中，`flags`中包含一些标识位，`prio`表示交换区的优先级，`swap_map`存放了每一个交换区上page的引用计数，为0表示该page是空闲的，可以换入。cluster相关的是针对SSD的一种优化，将相邻的多个page看作一个cluster，再用链表串起来，降低锁的竞争。

```c
struct swap_info_struct {
     unsigned int flags;
     spinlock_t sdev_lock;
     struct file *swap_file;
     struct block_device *bdev;
     struct list_head extent_list;
     int nr_extents;
     struct swap_extent *curr_swap_extent;
     unsigned old_block_size;
     unsigned short * swap_map;
     unsigned int lowest_bit;
     unsigned int highest_bit;
     unsigned int cluster_next;
     unsigned int cluster_nr;
     int prio;   /* swap priority */
     int pages;
     unsigned long max;
     unsigned long inuse_pages;
     int next;   /* next entry on swap list */
};
```

- `swap_map`：`swap_map`中的每一个`unsiged short`都表示一个page
- `lowest_bit`&`highest_bit`：表示可使用的交换区范围
- `max`：交换区的大小
交换区的第一个和最后一个页面通常不用于交换，而是存放一些管理信息（位图、可用数量等）。


```c
struct swap_list_t swap_list = {-1, -1};
struct swap_info_struct swap_info[MAX_SWAPFILES];
```

为了支持多文件交换设备，在内核中由`swap_info`数组和按照优先级排序的`swap_list`进行管理。当调用`swap_on`将一个文件用于页面交换后就会链入`swap_list`中。

那么内核是如何实现对在内存上的页面和在文件交换区上页面进行地址转换和查找的统一的呢。我们知道`pte_t`是一个32bit的unsigned整数，由于每一个page大小为4K，低12bit在`pte_t`中是作为flag来使用的，高20bit作为物理地址。一个页面在MMU中在检查flag时，bit0-present如果为0表示当前页面在交换区上，此时pte_t的内容就会按照`swp_entry_t`来进行翻译。

```c
typedef struct {
     unsigned long val;
} swp_entry_t;
```

`swp_entry_t`和`pte_t`的大小一致，但是`swp_entry_t`的format如下：

```c
[ 31:11][10:1][  0:0  ]
[offset][type][present]
```

- bits[31:11]为page在交换区内的偏移，对应着`swap_map`的下标；
- bits[10:1]表示具体的文件，对应着`swap_info`的下标。
- bits[0:0]始终为0表示page在交换区上；

此时内核就能够按照swap_entry_t去找到页面内容。至此，[内存页面管理](./01-linux-memory-manage-base.md)以及此文中的交换区内存管理就都已结束。

## 周转流程

在内核中一个页面的状态会随着分配、释放、交换不断的进行变化。
这里出现了两个概念：clean和active。

active好理解，当该物理页面至少有一个引用时就是active的。clean表示页面的内容没有被修改过，数据仍然有效。这两个flag会影响到页面的周转顺序。

```text

free --alloc-> active_list <-unmap/map-> inactive_dirty_list
  ^                 ^                           |
  |                 |                          swap
  |              unmap/map                      |
  |                 |                           v
  |                 |------------------> inactive_clean_list
  |                                             |
  |                                             |
  ---------------reclaim------------------------|
```

最简单的内存置换策略：当有内存需要分配时从空闲链表进行分配，内存不足时将一些页面swap到磁盘上，让出一些物理内存。这样的设计存在两个缺点，第一个时当内存不足时临阵磨枪影响实时性，好的方式应该是定期回收一部分内存，第二个缺点是刚换出的页面有可能再次被访问，产生大量的抖动。

因此，更好的设计是将页面的释放分为两部分做，第一步取消映射将页面加入inactive_dirty_list或者inactive_clean_list，然后如果出现了对取消映射页面的访问只需要重新建立映射就可以了，而在inactive_dirty_list队列中的页面也不是立刻写出，而是通过一段时间的老化以后才写出去变成clean页面。对于clean页面也不是立刻就返还给free-list，因为页面中的内容依然有效，还是可以等到非做不可时才进行回收，将一个clean页面变成free的开销是极小的，但是如果出现访问节省的磁盘IO开销就很关键了。核心就是推迟耗时的磁盘读写操作到非做不可的时候再做。

## Note

参考源码为 2.6
