# 伙伴系统API-概述

伙伴系统实现了对页帧的管理，并对外提供分配和释放的接口。所有的上层内存管理，比如vmalloc、slab分配器等都是通过伙伴系统暴露的接口申请一个或着多个页帧。

## 常用封装API

Linux内存管理多个Node，Node管理多个zone（DMA、NORMAL、HIGHMEM），每个zone管理多个自由链表组`struct free_area`（按照分配阶），每个free_area管理由多个的双向链表（按照UNMOVABLE、RECLIAM、MOVABLE、REVERSE等迁移类型），每个链表包含多个固定数量page构成的内存块。

整个Linux内存管理的体系结构就是伙伴系统，伙伴系统最底层管理是分配阶大小的内存块，因此提供分配接口也是以order为参数，以尝试分配`2^order`页。。

常见的伙伴系统API包括:

- `alloc_pages(mask, order)`: 分配指定分配阶的内存块并返回`struct page`指针指向连续页的首个页帧，`alloc_page(mask, order)`是order==0时的简化形式，只分配一页。
- `get_zeroed_page(mask)`: 分配一个页帧，并清零。
- `__get_free_pages(mask, order)`: 和`alloc_pages`的工作方式相同，但是返回的是**虚拟地址**而不是page指针。`__get_free_page(mask)`只分配一页。
- `get_dma_pages(gfp_mask, order)`: 获取ZONE_DMA的pages。

对应的释放接口有`free_page(strcut page *)`，`free_pages(struct page *, order)`，`__free_page(addr)`，`__free_pages(addr, order)`。前两个接收page指针作为参数，后两个接受虚拟地址作为参数，order指定负责释放固定分配阶的内存块。
