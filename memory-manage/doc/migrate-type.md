# 迁移类型

在伙伴系统中长时间的内存分配之后很容易造成内存碎片，即物理内存总量不少但是无法合并为大的连续内存块。而在现代CPU中提供了huge page的可能，可以分配超大块的page，在TLB中使用更少级的地址转换操作。一个page覆盖了更大的地址范围，大幅度的提高了TLB的命中概率。对于内存密集型应用来说使用巨页能够有效提高性能。huge page的分配自然需要连续的内存区域。

## 已分配Page类型

在Linux内核中已分配的page被划分为以下三种类型：

- 不可移动页：不可以移动的页，内核分配的Page属于该类型。
- 可回收页：该类型页不可以移动，但是可以回收（删除内容），其内容可以从某些源重新生成，比如文件映射。当内存中可回收页过多就会唤醒kswapd进行内存回收。
- 可移动页：可以任意移动，用户空间分配的内存属于该种类型，这种page通过页表建立映射，移动后只需要重新建立映射即可。

为了避免不可移动页出现在可移动页之间产生碎片导致大块内存不可使用，在内核启动时将所有伙伴系统中所有内存页进行预分配，每类内存分配一定的内存块，在内存分配时可以选择指定迁移类型的内存块。

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

## 总结

迁移类型对伙伴系统中自由链表的分类从分配机制上减少了出现页间内存碎片的概率。
