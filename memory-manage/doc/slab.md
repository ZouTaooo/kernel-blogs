# slab

slab分配器从伙伴系统获取页帧进行管理，对外提供小内存块的分配以，同时为一些特定内核数据结构的提供缓存。

slab分配器的为的是解决内核中小内存块的分配需求，毕竟伙伴系统的内存分配以page为单位实在太大了。

使用slab带来以下几个好处:

- 减少了伙伴系统的访问次数，小内存的分配在slab分析器中就可以完成分配，释放的小内存也会回到slab分配器中，并不返回给伙伴系统。加快了分配和释放速度，同时提高了分配的内存在内存中驻留的概率。
- 减少伙伴系统操污染cpu的数据和指令cache。
- 数据如果直接存储在伙伴系统提供的页中会出现对象地址经常处于二的幂次方附近，造成某些cache line成为热点，造成cache line的不均衡。slab提供了着色机制，以实现对象在cache line分布均匀。

## slob & slab & slub

slab分配器存在两个问题，在嵌入式系统上slab分配器的代码量和逻辑过于复杂，在超大计算机系统上slab分配器所需要的元数据会占用大量的内存。因此为了满足这两种场景的小内存分配需求内核提供了slob和slub分配器。

- slob: 使用最先适配优先算法+单链表实现，以满足代码量小+简单这个需求。
- slub: 为了减少元数据的空间占用，slub在`struct page`中的一些未使用的字段中存放信息，虽然增加了复杂性，但是确实能够在大型计算机上提供更好的性能。

虽然在实现上三种分配器存在区别，但是在对外提供的接口上却完全一致。

## slab使用接口

slab提供两种服务，第一种以`kmalloc&kfree`为入口，分配指定大小的小内存块。第二种提供对象缓存服务，但是这种使用方式需要手动创建缓存，相关API有
`kmem_cache_create`、`kmem_cache_alloc`、`kmem_cache_free`。

通过`cat /proc/slvbinfo`查看slab信息。

```c
[sudo] password for weizhen.zt: 
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
kcopyd_job             0      0   3312    9    8 : tunables    0    0    0 : slabdata      0      0      0
io                     0      0     64   64    1 : tunables    0    0    0 : slabdata      0      0      0
dm_uevent              0      0   2632   12    8 : tunables    0    0    0 : slabdata      0      0      0
dm_old_clone_request      0      0    320   25    2 : tunables    0    0    0 : slabdata      0      0      0
dm_rq_target_io        0      0    120   34    1 : tunables    0    0    0 : slabdata      0      0      0
                                                    ...
kmalloc-8192         122    140   8192    4    8 : tunables    0    0    0 : slabdata     35     35      0
kmalloc-4096         746    760   4096    8    8 : tunables    0    0    0 : slabdata     95     95      0
kmalloc-2048         906   1008   2048   16    8 : tunables    0    0    0 : slabdata     63     63      0
kmalloc-1024        3089   3472   1024   16    4 : tunables    0    0    0 : slabdata    217    217      0
                                                    ...
kmalloc-8           4608   4608      8  512    1 : tunables    0    0    0 : slabdata      9      9      0
kmem_cache_node      384    384     64   64    1 : tunables    0    0    0 : slabdata      6      6      0
kmem_cache           231    231    384   21    2 : tunables    0    0    0 : slabdata     11     11      0
```

`kmalloc`就会在命名为`kmalloc-size`中的slab分配器中分配内存块。而其他命名的slab分配器则是对应的一些内核对象slab缓存。


## slab实现

