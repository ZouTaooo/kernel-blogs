# 冷热页机制

在进行内存访问时的大概流程如下：

1. 由CPU发出访存指令
2. 地址转化，MMU根据页表转换或者通过TLB得到物理地址
3. 访问cache
4. 如果cache miss，访问物理内存读入cache

因此如果访问一个在cache中的物理内存能大幅度提高访问速度，在Linux中将内容仍在cache中page称为hot page。

## percpu_pageset_t

由于每个CPU都有自己的cache，而在在Linux中内存分配最终会落在Zone上，因此在zone中有一个per-cpu的`struct per_cpu_pageset`，per-zone的per-cpu的`struct per_cpu_pageset`就负责管理在CPU在该zone上进行分配时的冷热页。

```c
struct zone {
    struct per_cpu_pageset pageset[NR_CPUS];
}
```

相关数据结构如下，`per+cpu_pageset`中有一个`per_cpu_pages`成员，冷热页的实现就在`per_cpu_pages`中。冷热页通过一个链表表示，热页放在链表首部，冷页放在链表尾部。

- `count`: 冷热链表中的page个数。
- `high`: 当`count`超过high时说明缓存的冷热页过多，触发批量返回页帧给伙伴系统。
- `batch`: 批量移除或添加page的个数
- `list`: 冷热链表

```c
struct per_cpu_pages {
    int count;		/* number of pages in the list */
    int high;		/* high watermark, emptying needed */
    int batch;		/* chunk size for buddy add/remove */
    struct list_head list;	/* the list of pages */
};


struct per_cpu_pageset {
    struct per_cpu_pages pcp;
#ifdef CONFIG_NUMA
    s8 expire;
#endif
#ifdef CONFIG_SMP
    s8 stat_threshold;
    s8 vm_stat_diff[NR_VM_ZONE_STAT_ITEMS];
#endif
} ____cacheline_aligned_in_smp;

```

## 单页的分配

当某个zone符合分配条件时，`buffered_rmqueue`就会**尝试**从该zone中分配内存。

首先`buffered_rmqueue`会检查`gfp_flags`是否标记了`__GFP_COLD`，`!!(gfp_flags & __GFP_COLD)`中两个`!!`的作用是将cold的值限定为0或者1。根据gfp_flags确定迁移类型`migratetype`，迁移类型与内存碎片管理相关。

冷热页主要优化的是单个页面的分配和释放，也就是当分配阶为0的情况。首先找到当前zone上的当前cpu的冷热链表`pcp`，检查`pcp`是否有缓存的page，如果没有则调用`rmqueue_bulk`从当前zone上分配`pcp->batch`个pages加入到冷热链表中。如果存在pages并且cold为1说明优先分配冷页，从后往前遍历，否则从前往后遍历，找到第一个满足迁移类型的page返回，可以看到冷热的概念只表示被返还的顺序，后进入的链表的页帧相对热一些，分配并不能真的保证分配的hot page在cache中，只是可能性更大。

`&page->lru == &pcp->list`这个判断表示在冷热链表中未找到满足迁移类型的page。此时则调用`rmqueue_bulk`后，再尝试分配一个page。

```c
static struct page *buffered_rmqueue(struct zone *preferred_zone,
            struct zone *zone, int order, gfp_t gfp_flags)
{
    unsigned long flags;
    struct page *page;
    int cold = !!(gfp_flags & __GFP_COLD);
    int cpu;
    int migratetype = allocflags_to_migratetype(gfp_flags);

again:
    cpu  = get_cpu();
    if (likely(order == 0)) {
        struct per_cpu_pages *pcp;

        pcp = &zone_pcp(zone, cpu)->pcp;
        local_irq_save(flags);
        // 如果冷热链表为空 从zone中分配负责迁移类型的pages加入冷热链表
        if (!pcp->count) {
            pcp->count = rmqueue_bulk(zone, 0,
                    pcp->batch, &pcp->list, migratetype);
            if (unlikely(!pcp->count))
                goto failed;
        }

        // 如果是要求分配冷页则从后往前查找，否则从前往后查找，找到第一个满足迁移类型的page
        /* Find a page of the appropriate migrate type */
        if (cold) {
            list_for_each_entry_reverse(page, &pcp->list, lru)
                if (page_private(page) == migratetype)
                    break;
        } else {
            list_for_each_entry(page, &pcp->list, lru)
                if (page_private(page) == migratetype)
                    break;
        }

        // 如果没有合适的page 此时回到了开头 尝试再次分配符合迁移类型的pages加入列表后从链表中分配page
        /* Allocate more to the pcp list if necessary */
        if (unlikely(&page->lru == &pcp->list)) {
            pcp->count += rmqueue_bulk(zone, 0,
                    pcp->batch, &pcp->list, migratetype);
            page = list_entry(pcp->list.next, struct page, lru);
        }

        list_del(&page->lru);
        pcp->count--;
    } 
...
}
```

## 单页的释放

在内核中释放page的基础API是`__free_pages`，`__free_pages`会检查释放的页的分配阶是否为0，如果为0则调用`free_hot_page`。`free_hot_page`则会调用`free_hot_cold_page`携带参数表示释放的是hot-page。

```c
void __free_pages(struct page *page, unsigned int order)
{
    if (put_page_testzero(page)) {
        if (order == 0)
            free_hot_page(page);
        else
            __free_pages_ok(page, order);
    }
}

void free_hot_page(struct page *page)
{
    free_hot_cold_page(page, 0);
}
    
void free_cold_page(struct page *page)
{
    free_hot_cold_page(page, 1);
}

```

`free_hot_cold_page`中和冷热链表相关的代码如下，cold参数决定了是将page链入链表的首部还是尾部，然后设置page的`private`字段为page的迁移类型。

此时如果冷热链表的页总数`count`超过了`high`，此时调用`free_pages_bulk`返回`batch`个pages给伙伴系统。这种方式被称为**惰性合并**，能够避免大量不必要的合并操作（合并了又被分配的情况）。

```c
static void free_hot_cold_page(struct page *page, int cold)
{
    struct zone *zone = page_zone(page);
    struct per_cpu_pages *pcp;
    unsigned long flags;

    ...

    pcp = &zone_pcp(zone, get_cpu())->pcp;
    if (cold)
        list_add_tail(&page->lru, &pcp->list);
    else
        list_add(&page->lru, &pcp->list);
    set_page_private(page, get_pageblock_migratetype(page));
    pcp->count++;
    if (pcp->count >= pcp->high) {
        free_pages_bulk(zone, pcp->batch, &pcp->list, 0);
        pcp->count -= pcp->batch;
    }
    ...
}
```

## 总结

冷热页机制有以下优点：

- free的page加入热页后，一段时间内如果再次访问大概率只需要重新建立映射，而不需要将内容读取到cache中，访问速度更快。
- 通过从伙伴系统批量分配和释放pages，减少了使用伙伴系统的次数，因此降低了伙伴系统的分裂合并的次数，同时分配速度更快。
- 采取per-cpu的设计，区分了不同CPU的cache，同时避免了CPU同时分配内存时的竞争。
