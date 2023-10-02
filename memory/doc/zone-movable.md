# ZONE_MOVABLE

ZONE_MOVABLE是一个虚拟内存域，ZONE_MOVABLE内存区域的范围实际上会覆盖高端内存或者NORMAL内存。

ZONE_MOVABLE有两个作用，其一是解决内存碎片问题，将内存域分为可移动和不可移动的，避免不可移动内存处于可移动内存块之间。其二是用于虚拟内存的热插拔，容器动态的进行内存的缩容和扩容需要一整块内存的重分配，对于内存碎片的问题很敏感。

## ZONE_MOVABLE的开启

`ZONE_MOVABLE`的开启与是否指定了`kernelcore`有关，该参数指定了不可移动的内存数量，计算结果会存储在`required_kernelcore`。`movablecore`指定了可移动的内存数量。如果同时指定`kernelcore`和`movablecore`此时会按照两种方式计算出较大的`required_kernelcore`，如果都未指定，则该区域范围为0，相当于未开启。`find_zone_movable_pfns_for_nodes`就是在做这个事情，按照参数计算出`ZONE_MOVABLE`内存域的首个page的PFN（page frame number）。

一直都说`ZONE_MOVABLE`是一个虚拟内存域，是因为`ZONE_MOVABLE`管理了NORMAL或者HIGHMEM的内存，看到这里之后我就产生了疑问，管理的内存与其他zone是重叠的吗？答案是不同内存域的内存是不重叠的，按照计算出来的`ZONE_MOVABLE`在每个node上的起始pfn就已经确定了`ZONE_MOVABLE`管理的内存区域，后续在做其他内存区域的初始化时就会对原本的内存区域进行调整，`adjust_zone_range_for_zone_movable`就在做这个事情，如果别的内存区域计算出的范围和`ZONE_MOVABLE`产生了重叠，就会对这些区域进行压缩，比如正常情况下的HIGHMEM区域全部都在`ZONE_MOBVABLE`中了，调整后该内存区域管理的内存就为0.

```c
void __meminit adjust_zone_range_for_zone_movable(int nid,
     unsigned long zone_type,
     unsigned long node_start_pfn,
     unsigned long node_end_pfn,
     unsigned long *zone_start_pfn,
     unsigned long *zone_end_pfn)
{
 /* Only adjust if ZONE_MOVABLE is on this node */
 if (zone_movable_pfn[nid]) {
  /* Size ZONE_MOVABLE */
  if (zone_type == ZONE_MOVABLE) {
   *zone_start_pfn = zone_movable_pfn[nid];
   *zone_end_pfn = min(node_end_pfn,
    arch_zone_highest_possible_pfn[movable_zone]);

  /* Adjust for ZONE_MOVABLE starting within this range */
  } else if (*zone_start_pfn < zone_movable_pfn[nid] &&
    *zone_end_pfn > zone_movable_pfn[nid]) {
   *zone_end_pfn = zone_movable_pfn[nid];

  /* Check if this whole range is within ZONE_MOVABLE */
  } else if (*zone_start_pfn >= zone_movable_pfn[nid])
   *zone_start_pfn = *zone_end_pfn;
 }
}
```

## 访问ZONE_MOVABLE

我们知道访问内存的入口函数需要指定一些GFP_MASK，其中和访问内存区域相关的会将GFP_MASK转化为`ZONE_TYPE`，仅当同时设置`__GFP_HIGHMEM | __GFP_MOVABLE`时才会对ZONE_MOVABLE内存区域进行访问。其余部分和正常从伙伴系统分配释放内存无异。

```c
static inline enum zone_type gfp_zone(gfp_t flags)
{
#ifdef CONFIG_ZONE_DMA
    if (flags & __GFP_DMA)
        return ZONE_DMA;
#endif
#ifdef CONFIG_ZONE_DMA32
    if (flags & __GFP_DMA32)
        return ZONE_DMA32;
#endif
    if ((flags & (__GFP_HIGHMEM | __GFP_MOVABLE)) ==
            (__GFP_HIGHMEM | __GFP_MOVABLE))
        return ZONE_MOVABLE;
#ifdef CONFIG_HIGHMEM
    if (flags & __GFP_HIGHMEM)
        return ZONE_HIGHMEM;
#endif
    return ZONE_NORMAL;
}
```
