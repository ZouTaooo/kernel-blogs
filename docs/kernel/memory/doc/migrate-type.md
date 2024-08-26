# 迁移类型与内存碎片

## 前言

在伙伴系统中长时间的内存分配之后很容易造成内存碎片，即物理内存总量不少但是无法合并为大的连续内存块。而在现代CPU中提供了huge page的可能，可以分配超大块的page，在TLB中使用更少级的地址转换操作。一个page覆盖了更大的地址范围，大幅度的提高了TLB的命中概率。对于内存密集型应用来说使用巨页能够有效提高性能。huge page的分配自然需要连续的内存区域。因此避免内存碎片对于需要分配大块内存分配的场景来说非常重要。
**Note：这里的内存碎片指的以页为粒度的碎片，迁移类型是在伙伴系统层面避免碎片产生的一种机制。**

## 已分配Page类型

在Linux内核中将已分配的page被划分为以下三种类型：

- 不可移动页：不可以移动的页，内核分配的Page属于该类型，因为内核分配的Page的页表采取的是直接映射，这些页移动后无法通过更改页表映射来进行重定向。
- 可回收页：该类型页不可以移动，但是可以回收（删除内容），其内容可以从某些源重新生成，比如文件映射。当内存中可回收页过多就会唤醒kswapd进行内存回收。
- 可移动页：可以任意移动，用户空间分配的内存属于该种类型，这种page通过页表建立映射，移动后只需要重新建立映射即可。

为了避免不可移动页出现在可移动页之间，导致大块内存不可使用，在内核启动时将所有伙伴系统中所有内存页划分为了不同的迁移类型，每类内存包含一定数量的页，在内存分配时可以选择指定迁移类型的内存块，这样就不会出现不可移动的页出现在了可移动页的内部，导致无法通过页面迁移来进行内存规整。

## 伙伴系统和迁移类型

在伙伴系统中每个分配阶的自由链表都有多个，`struct free_area`中的每个自由链表对应一种迁移类型。

```c
struct zone {
    struct free_area free_area[MAX_ORDER];
}

struct free_area {
    struct list_head free_list[MIGRATE_TYPES];
    unsigned long  nr_free;
};
```

内核支持的迁移类型包括以下五种，`MIGRATE_UNMOVABLE`表示不可移动，`MIGRATE_RECLAIMABLE`对应可回收，`MIGRATE_MOVABLE`对应可移动，`MIGRATE_RESERVE`对应保留内存，这部分用于紧急情况的内存分配。`MIGRATE_ISOLATE`无法分配。

```c
#define MIGRATE_UNMOVABLE     0
#define MIGRATE_RECLAIMABLE   1
#define MIGRATE_MOVABLE       2
#define MIGRATE_RESERVE       3
#define MIGRATE_ISOLATE       4 /* can't allocate from here */
#define MIGRATE_TYPES         5
```

当内存分配在指定的迁移类型列表中无法完成分配时会按照备选列表的顺序选择其他迁移类型的自由链表继续内存分配。

```c
static int fallbacks[MIGRATE_TYPES][MIGRATE_TYPES-1] = {
    [MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE,   MIGRATE_RESERVE },
    [MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE,   MIGRATE_RESERVE },
    [MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_RESERVE },
    [MIGRATE_RESERVE]     = { MIGRATE_RESERVE,     MIGRATE_RESERVE,   MIGRATE_RESERVE }, /* Never used */
};
```
这里有一个问题是，为什么备选列表的分配顺序是这样的？首先`MIGRATE_RESERVE`不必多说，这种类型的内存非必要情况下不分配，因此始终处于备选最后一个选项。而`MIGRATE_UNMOVABLE`和`MIGRATE_RECLAIMABLE`分配失败时会偏向于靠后分配可移动内存，这是我们这个机制的目标要求的，避免在可移动内存中引入不可移动的内存，减少内存碎片的产生。而`MIGRATE_MOVABLE`分配失败时会优先从可回收的内存中分配，我的理解是不可移动内存的数量更稀缺，而可回收内存内存由于其特性，可以反复使用紧急程度更低。
## 总结

迁移类型对伙伴系统中的自由链表进行分类，从分配机制上减少了出现页间内存碎片的概率，但是并不是完全避免了。毕竟内存分配成功是第一要务，当内存不足时依然会引入内存碎片。
