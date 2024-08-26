# 伙伴系统API概述

## 前言

伙伴系统实现了对页帧的管理，并对外提供分配和释放的接口。所有的上层内存分配，比如`vmalloc`、slab分配器等都是通过伙伴系统暴露的接口申请一个或者多个连续页帧。

## 伙伴系统API

Linux内存管理的最上层包含多个Node，Node管理多个Zone（DMA、NORMAL、HIGHMEM等），每个Zone又管理多个自由链表组`struct free_area`（按照分配阶划分），每个`free_area`又管理着多个双向链表（按照UNMOVABLE、RECLIAM、MOVABLE、REVERSE等迁移类型划分），每个链表管理着分配阶相同的内存块。

Linux物理内存管理的基础是伙伴系统，伙伴系统管理的是不同分配阶的内存块，因此提供分配接口也是以分配阶`order`为参数，以尝试分配一个包含`2^order`个连续页帧的内存块。

常见的伙伴系统API包括:

- `alloc_pages(mask, order)`: 分配指定分配阶的内存块并返回`struct page`指针指向连续页的首个页帧，`alloc_page(mask)`对`alloc_pages()`进行封装，尝试申请一个分配阶为0的内存块。
- `get_zeroed_page(mask)`: 分配一个清零后页帧。
- `__get_free_pages(mask, order)`: 和`alloc_pages()`的工作方式相同，但是返回的是虚拟地址而不是`struct page`指针。`__get_free_page(mask)`和`alloc_page()`类似，尝试分配一个清零的页帧。
- `get_dma_pages(gfp_mask, order)`: 申请`ZONE_DMA`内存域的内存块。

对应的释放接口有：

- `free_page(strcut page *)`
- `free_pages(struct page *, order)`
- `__free_page(addr)`
- `__free_pages(addr, order)`

前两个释放接口函数接收`struct page`指针作为参数，后两个接受虚拟地址作为参数，`order`指定被释放内存块的分配阶。
