# 伙伴系统（六）内存分配的快速分配阶段

## 快速分配

```c
// mm/page_alloc.c
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order, int preferred_nid,
							nodemask_t *nodemask)
{
	unsigned int alloc_flags = ALLOC_WMARK_LOW;
	gfp_t alloc_mask;
	struct alloc_context ac = { };

	if (!prepare_alloc_pages(gfp_mask, order, preferred_nid, nodemask, &ac, &alloc_mask, &alloc_flags))
		return NULL;
	alloc_flags |= alloc_flags_nofragment(ac.preferred_zoneref->zone, gfp_mask);
	// 快速分配阶段
    page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
    ......
}
EXPORT_SYMBOL(__alloc_pages_nodemask);
```

在快速分配阶段内核采取更保守的分配策略，可以达到更好的内存访问效率，减少内存碎片。`get_page_from_freelist()`这个函数在实现上与分配策略完全解耦，是一个通用的内存分配函数，在慢速分配阶段也是使用该函数进行内存分配，但是会使用更为激进的内存分配策略来提高分配的成功率。

在快速分配阶段的分配策略初始化如下：

- 使用`ALLOC_WMARK_LOW`，按照Low水位线要求进行检查
- `prepare_alloc_pages()`: 初始化`ac`参数，确定用于搜索的Zone列表、搜索的起点、限制Node范围、限制Zone的范围（Moveable、High、Normal、DMA），在`get_page_from_freelist()`中基于以上信息可以遍历所有初步符合要求的Zone。设定内存的迁移类型，避免不同可移动性的内存混用产生内存碎片。如果有标记`__GFP_WRITE`，需要设置`spread_dirty_pages`为`true`，在快速分配阶段时要考虑Node的脏页分布情况。

还有一些更多的特性，比如`ALLOC_NOFRAGMENT`、`ALLOC_CPUSET`、`ALLOC_CMA`和`ALLOC_KSWAPD`，这些特性在一些特定场景也会设定，这里不详细展开。

## get_page_from_freelist

`get_page_from_freelist()`是一个通用的内存分配函数，该函数会遍历`zonelist`找到第一个符合要求的同时也是最合适的Zone并尝试从其中分配内存块。

代码如下主要包括两个部分，`for_next_zone_zonelist_nodemask`宏遍历`zonelist`期望找到一个满足条件的Zone，当找到了一个Zone时，进入`try_this_zone`代码块尝试从Zone中分配内存块，如果分配失败则继续寻找下一个有效的Zone。`for_next_zone_zonelist_nodemask`这个宏能找到每一个满足以下两个条件的Zone：

- Zone所属的Node在`ac->nodemask`中
- Zone在Node中的索引小于等于`ac->highest_zoneidx`

```c
static struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
						const struct alloc_context *ac)
{
	struct zoneref *z;
	struct zone *zone;
	struct pglist_data *last_pgdat_dirty_limit = NULL;
	bool no_fallback;

retry:
	for_next_zone_zonelist_nodemask(zone, z, ac->highest_zoneidx,
					ac->nodemask) {
		// 1. 检查Zone是否满足条件
try_this_zone:
		// 2. 尝试从Zone中分配内存
	}

	return NULL;
}
```

对于初步满足要求的Zone还要进行三个部分的检查，这里只简述核心的逻辑，其中有很多的小细节不进行展开：

- `cpuset`限制: 在一些场景下进程被限制只能使用固定Node上的内存，此时需要检查Zone所在的Node是否满足要求。但是在一些特殊情况下可以忽略该限制，比如此时处于中断上下文（不与具体的进程关联），或者是当前进程是一个OOM受害者（获得内存是为了更快释放更多的内存）。
- 脏页限制: 为了避免脏页集中在某一个Node上，检查Zone所在的Node脏页数量是否已经达到了上限。
- 水位限制: 检查Zone上的内存压力是否满足要求。

如果找到一个合适的Zone，则尝试从Zone中分配内存块。内核中单页的分配最频繁，针对单页分配内核维护了一个PER-CPU的缓存链表，缓解锁的竞争，提高分配效率。如果分配阶不为0则从伙伴系统中查找满足要求的最小内存块，如果最小的内存块的分配阶高于期望分配的分配阶则需要进行`expand`，将高分配阶的内存块拆分为多个低分配阶的内存块。
