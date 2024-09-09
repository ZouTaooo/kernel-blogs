# meminfo

`Documentation/filesystem/proc.rst`对`meminfo`的描述如下:
> Provides information about distribution and utilization of memory.  This
> varies by architecture and compile options.  

`/proc/meminfo`以各种维度对内存的使用和分布情况进行了统计，用于查看内存使用情况的`free`命令就是基于这个接口文件的数据进行加工的。

## 输出解析

下面的输出来自于Linux v4.19，没有开启`CONFIG_HIGHMEM`、透明巨页的等特性，没有开启内存交换。

```bash
MemTotal:       16130120 kB # 所有的可用内存，RAM总量减去了一些保留位和内核的二进制代码
MemFree:          293432 kB # 所有未分配的内存
MemAvailable:   14905192 kB # 空闲内存加上在保障内核稳定运行的情况下可以回收的内存，是一个估计值。
                            # 大概是MemFree+SReclaimable+Active(file)+Inactive(file)的基础上考虑每个内存域的low水线。
Buffers:          273432 kB # 磁盘块的临时存储，不会很大，一般在20MB左右
Cached:         12240900 kB # 内存中的PageCache大小，不包括SwapCached
SwapCached:            0 kB # 换出后又换入到的内存，但是内容仍然在swapfile中，这类内存如果要换出不需要重复写回swapfile，可以减少IO。
Active:          9252804 kB # 使用比较活跃的内存，非必要情况下不会回收，Active(anon) + Active(file) 
Inactive:        3884896 kB # 使用不过活跃的内存，更适合被回收用于其他用途，Inactive(anon) + Inactive(file)
Active(anon):     598912 kB # 活跃的匿名页
Inactive(anon):      344 kB # 不活跃的匿名页
Active(file):    8653892 kB # 活跃的PageCache
Inactive(file):  3884552 kB # 不活跃的PageCache
Unevictable:           0 kB # 无法回收的内存
SwapTotal:             0 kB # 交换区的大小
SwapFree:              0 kB # 已经在从RAM中置换到交换区的内存
Dirty:              1116 kB # 等待写回磁盘的内存
Writeback:             0 kB # 正在写回磁盘的内存
AnonPages:        603804 kB # 映射到用户空间页表的匿名页
Mapped:           346480 kB # 映射到用户空间页表的文件
Shmem:               812 kB # shmem和tmpfs使用的内存
Slab:            2444456 kB # 内核内部的数据结构缓存
SReclaimable:    2410472 kB # Slab中可回收的部分
SUnreclaim:        33984 kB # Slab中不可回收的部分
KernelStack:        7712 kB # 内核栈使用的内存
PageTables:        10008 kB # 最底层页表使用的内存
```

## 内存的分类

* 按照是否已分配分类：所有存放在伙伴系统的中的空闲列表中页帧都属于未被分配的内存，对应`MemFree`的部分。已分配的内存则会用于slab分配器、用户态进程、共享内存、PageCache等多种用途。
* 按照活跃度分类：内核使用LRU来管理已分配的内存页，将所有内存分为`Active`和`Inactive`两类，内存有压力时优先从不活跃的链表中回收页面。
* 按匿名性分类：如果一个页面存放的内容来自于文件，这个页面则是非匿名内存，反之亦然。这两类内存在回收上有较大的区别，在开启了内存交换的情况下，匿名内存可以写入交换区后释放，非匿名内存脏页改则直接写回文件后释放，非匿名干净页则直接释放。而在未开启内存交换的情况下，匿名内存无法被释放回收，而非匿名内存则可以将脏页刷盘后释放和回收。
* 按共享性分类：如果一个页面被多个进程共享，则该页面属于共享内存，共享内存除了包括mmap映射的内存以外还包括tmpfs。

## MemAvailable

可用内存总量`MemAvailable`是内核估算的一个值，这个值表示在不影响内核稳定运行的情况下（不触发内存回收）可以给用户态进程使用的内存总量。参考`si_mem_available()`函数，详细的计算公式如下：

```bash
MemAvailable = MemFree - totalreserve_pages + max(pagecache / 2, pagecache - wmark_low) + max(SReclaimable / 2, SReclaimable - wmark_low) + INDIRECTLY_RECLAIMABLE
```

其中：

* `totalreserve_pages`：内核预留的内存
* `wmark_low`：每个Zone的low内存水线之和
* `pagecache`：`Inactive(file)` + `Active(file)`
* `SRebclaimable`：Slab中可回收的部分
* `INDIRECTLY_RECLAIMABLE`：`/* Part of the kernel memory, which can be released under memory pressure.`注释上看是内存有压力时可以释放的部分内核内存。

但是一般来说`wmark_low`都比较小，简化版公式如下：

```bash
MemAvailable = MemFree - totalreserve_pages + Inactive(file) + Active(file) + SReclaimable
```
