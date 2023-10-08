<!-- # 伙伴系统（一）基本结构 -->
## 前言

伙伴系统是Linux内存管理的精华部分，所有的物理内存管理使用的都是该机制。伙伴系统保障了内存分配的速度和效率，同时思想和实现又相当简单。这里对伙伴系统的结构进行简要的介绍。

## 伙伴系统结构

在zone中有一个成员`free_area`是伙伴系统的实现的关键部分，`free_area`是一个`struct free_area`数组。

每一个`struct free_area`管理了**一组**自由链表**组**，按照分配阶进行划分。每个自由链表组又按照迁移类型划分为了多个链表，这里的迁移类型与内存碎片机制相关，与伙伴系统的核心无关，可以暂时视为管理了一个链表。该链表管理着大小相同的内存块，每个内存块由固定数目的连续page组成，内存块的大小和`struct free_area`在数组中的偏移有关，偏移代表了内存块的分配阶。`nr_free`表示自由链表中的内存块个数。因此每个自由链表管理的pages总数为`nr_free * (1 << order)`。

```c
struct zone {
    struct free_area free_area[MAX_ORDER];
}

struct free_area {
    struct list_head	free_list[MIGRATE_TYPES];
    unsigned long		nr_free;
};

```

`MAX_ORDER`表示系统支持的最大分配阶，通常情况下是11，表示伙伴系统支持分配的最大内存块为`1 << 10 = 1024`个pages。在一些特定系统架构下该值也可以被修改。

```c
#ifndef CONFIG_FORCE_MAX_ZONEORDER
#define MAX_ORDER 11
#else
#define MAX_ORDER CONFIG_FORCE_MAX_ZONEORDER
#endif
#define MAX_ORDER_NR_PAGES (1 << (MAX_ORDER - 1))
```

伙伴系统分配的内存块超过需要的内存块时会将该内存块切分成多个较小的内存块放入对应的自由链表中。

page回收时可以根据内存块的地址推算出伙伴的地址，可以检查伙伴是否也处于空闲状态从而触发合并。
