# 复合页

在从伙伴系统进行内存分配时有一个`__GFP_COMP`分配标记，该标记表示从伙伴系统分配的连续页帧为一个复合页。

复合页可以用于hugetlb，减少tlb中地址转化的次数，减少tlb miss几率，同时提高tlb的地址转化速度。这里我对huge page的具体用途以及hugetlb的使用暂且不谈，先描述一下复合页在内核中是如何实现、分配以及释放的。

## 复合页的设计

复合页由连续的多个page构成，其中首个page称为head page，剩余的page都称作tail page。

![Alt text](../imgs/compound.png)

所有的page的`first_page`字段都存放指向head page的指针（该图来自于深入Linux内核架构，和code中的实现略有区别），此外head page后的第一个page的lru.next和lru.prev分别存放释放复合页的函数指针以及复合页的分配阶。

head page的lru.next和lru.prev自然用于放入某个队列进行管理。

此外，所有的page的flag都被标记上PG_compound。

## 复合页的分配

复合页的分配和普通的伙伴系统内存分配相同，区别在于做了额外的初始化操作。

```c
static int prep_new_page(struct page *page, int order, gfp_t gfp_flags)
{
    ...
    if (order && (gfp_flags & __GFP_COMP))
        prep_compound_page(page, order);
    ...
}
```

`set_compound_page_dtor`设置好析构函数用于释放复合页，存放位置为`page[1].lru.next`。

`set_compound_order`设置复合页的分配阶，存放位置为`page[1].lru.prev`。

然后为head page以及每个tail page设置好PG_flag,head page的flags会添加`PG_compound`，而tail page会同时设置`PG_reclaim`和`PG_compound`，用以区分head page和tail page。

```c
static void prep_compound_page(struct page *page, unsigned long order)
{
    int i;
    int nr_pages = 1 << order;

    set_compound_page_dtor(page, free_compound_page);
    set_compound_order(page, order);
    __SetPageHead(page);
    for (i = 1; i < nr_pages; i++) {
        struct page *p = page + i;

        __SetPageTail(p);
        p->first_page = page;
    }
}
```

## 复合页的释放

首先释放复合页需要从page指针的`first_page`找到head page，然后从第一个tail page的`lru.next`上获取析构函数，`lru.prev`上获取分配阶，调用析构函数即可。

```c
static void put_compound_page(struct page *page)
{
    page = compound_head(page);
    if (put_page_testzero(page)) {
        compound_page_dtor *dtor;

        dtor = get_compound_page_dtor(page);
        (*dtor)(page);
    }
}
```
