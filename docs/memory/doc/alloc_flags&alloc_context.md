# 内存分配-策略和上下文

## 内存分配的准备阶段

在内存分配的准备阶段要完成两个任务，对于不合法的请求在进入分配阶段前返回分配失败，对于合法的请求要为快速分配阶段设置好分配上下文信息，准备阶段的代码如下：

```c
// mm/page_alloc.c
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order, int preferred_nid,
							nodemask_t *nodemask)
{
	unsigned int alloc_flags = ALLOC_WMARK_LOW;
	gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
	struct alloc_context ac = { };

	if (unlikely(order >= MAX_ORDER)) {
		WARN_ON_ONCE(!(gfp_mask & __GFP_NOWARN));
		return NULL;
	}

	gfp_mask &= gfp_allowed_mask;
	alloc_mask = gfp_mask;
	if (!prepare_alloc_pages(gfp_mask, order, preferred_nid, nodemask, &ac, &alloc_mask, &alloc_flags))
		return NULL;
    ...
}
EXPORT_SYMBOL(__alloc_pages_nodemask);
```

- 分配阶参数校验: 对`order`进行检验，`MAX_ORDER`的默认值为`11`，内核支持最大的分配阶是`MAX_ORDER - 1`，对于超过最大分配阶的内存分配请求会直接返回分配失败。
- 内存分配上下文准备: `prepare_alloc_pages()`识别分配请求的要求，并且为快速内存分配设置上下文信息。

## alloc_flags

`alloc_flags`变量有控制内存分配行为、限制内存水位等用途。其中各个Bit的定义如下：

```c
#define ALLOC_WMARK_MIN		WMARK_MIN
#define ALLOC_WMARK_LOW		WMARK_LOW
#define ALLOC_WMARK_HIGH	WMARK_HIGH
#define ALLOC_NO_WATERMARKS	0x04 /* don't check watermarks at all */
#define ALLOC_WMARK_MASK	(ALLOC_NO_WATERMARKS-1)

#ifdef CONFIG_MMU
#define ALLOC_OOM		0x08
#else
#define ALLOC_OOM		ALLOC_NO_WATERMARKS
#endif

#define ALLOC_HARDER		 0x10 /* try to alloc harder */
#define ALLOC_HIGH		 0x20 /* __GFP_HIGH set */
#define ALLOC_CPUSET		 0x40 /* check for correct cpuset */
#define ALLOC_CMA		 0x80 /* allow allocations from CMA areas */

#ifdef CONFIG_ZONE_DMA32
#define ALLOC_NOFRAGMENT	0x100 /* avoid mixing pageblock types */
#else
#define ALLOC_NOFRAGMENT	  0x0
#endif

#define ALLOC_KSWAPD		0x800 /* allow waking of kswapd, __GFP_KSWAPD_RECLAIM set */
```

- `ALLOC_WMARK_MIN`，`ALLOC_WMARK_LOW`, `ALLOC_WMARK_HIGH`，`ALLOC_NO_WATERMARKS`: 内存分配过程需要对Zone的水位线进行检查，这四个标志位表明不同的水线的要求，会影响到分配是否成功。
- `ALLOC_OOM`: 表明当前任务是一个OOM受害者，在OOM进程退出的过程中需要更努力的分配内存，以便可以释放更多的内存。
- `ALLOC_HARDER`: 表示可以更努力的分配内存，可以进行更严格的回收、规整，降低水位线要求等等。
- `ALLOC_HIGH`: 表明是一个高优先级的内存分配请求。
- `ALLOC_CPUSET`: 表明需要进行亲和性检查。
- `ALLOC_CMA`: 允许从CMA（Contiguous Memory Allocator）区域进行内存分配
- `ALLOC_NOFRAGMENT`: 当请求希望分配的迁移类型的内存不足时，内核允许从其他迁移类型的自由链表中偷取内存，但是这样有可能会产生碎片，原本连续的内存块由于被切分为了不同的迁移类型难以进行合并。为了减少碎片，`ALLOC_NOFRAGMENT`要求在偷取时必须偷完整的内核支持的最大分配阶的内存块。
- `ALLOC_KSWAPD`: 允许唤醒`kswapd`，和`__GFP_KSWAPD_RECLAIM`的值一样。

## 分配上下文

`struct alloc_context`存放了内存分配时的上下文，`prepare_alloc_pages()`会分析GFP_MASK并设置上下文。

```c
struct alloc_context {
	struct zonelist *zonelist;
	nodemask_t *nodemask;
	struct zoneref *preferred_zoneref;
	int migratetype;
	enum zone_type highest_zoneidx;
	bool spread_dirty_pages;
};
```

各个成员变量的作用如下:

- `zonelist`: 有效Zone的搜索列表，每个Node有两个列表，如下方代码所示。在NUMA架构下，如果GFP_MASK如果设置了`__GFP_THISNODE`场景，此时使用无备选的列表`node_zonelists[ZONELIST_NOFALLBACK]`，表明只在当前Node包含的Zone中搜索，另一个是涵盖了所有的Node和Zone的列表`node_zonelists[ZONELIST_FALLBACK]`。

```c
// include/linux/mmzone.h
enum {
	ZONELIST_FALLBACK,	/* zonelist with fallback */
#ifdef CONFIG_NUMA
	/*
	 * The NUMA zonelists are doubled because we need zonelists that
	 * restrict the allocations to a single node for __GFP_THISNODE.
	 */
	ZONELIST_NOFALLBACK,	/* zonelist without fallback (__GFP_THISNODE) */
#endif
	MAX_ZONELISTS
};
```

- `nodemask`: 可选Node的掩码
- `preferred_zoneref`: 列表中第一个可用Zone的引用，在遍历时作为搜索的起点
- `migratetype`: 内存块的迁移类型要求，限制自由链表的可选范围
- `highest_zoneidx`: 最高Zone的索引，限制Zone的可选范围
- `spread_dirty_pages`: 是否要求脏页均匀分布
