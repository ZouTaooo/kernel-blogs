# 内存分配-基本流程

## __alloc_pages_nodemask

```c
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order, int preferred_nid,
							nodemask_t *nodemask);
```

内核为不同的内存分配场景封装了多个有多个内存分配接口，其中内存分配的核心的代码在`__alloc_pages_nodemask()`中，该函数按照分配策略从满足要求的Node和Zone中分配内存。作为一个通用的内存分配函数，接受以下四个参数：

- `gfp_mask`: GFP_MASK设定一些内存分配的限制和要求
- `order`: 分配的内存块的分配阶
- `preferred_nid`: 指定内存分配的首选Node
- `nodemask`: 指定可选的Node

## 关键流程

`__alloc_pages_nodemask()`的关键代码如下（去除了和关键流程无关的代码）:

```c
// mm/page_alloc.c
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order, int preferred_nid,
							nodemask_t *nodemask)
{
	struct page *page;
	unsigned int alloc_flags = ALLOC_WMARK_LOW;
	gfp_t alloc_mask;
	struct alloc_context ac = { };

	if (!prepare_alloc_pages(gfp_mask, order, preferred_nid, nodemask, &ac, &alloc_mask, &alloc_flags))
		return NULL;
	
    page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
	if (likely(page))
		goto out;
	
    page = __alloc_pages_slowpath(alloc_mask, order, &ac);
out:
	return page;
}
EXPORT_SYMBOL(__alloc_pages_nodemask);
```

一次内存分配有三个阶段:

- 准备阶段: `prepare_alloc_pages()`在准备阶段对输入的参数进行检查，对于一些无法完成的分配提前返回，如果是一次合法的内存分配请求则设置好内存分配的上下文，为快速内存分配做好准备。
- 快速内存分配阶段: `get_page_from_freelist()`会在分配策略GFP_MASK、水位要求`alloc_flags`和内存分配上下文`ac`的共同限制下进行内存分配。相比于慢速内存分配，在这个阶段的内存分配有更多考量和限制，比如要考虑内存碎片问题、NUMA内存亲和性、水位压力、脏页分布等等，绝大多数内存请求在这个阶段就能够完成分配。
- 慢速内存分配阶段: 如果在快速内存分配阶段无法完成内存分配说明此时内存出现了一定程度的紧缺，此时`__alloc_pages_slowpath()`会调整内存的分配策略来提高分配成功的概率，必要情况还会触发内存回收、内存规整、OOM killer等操作回收内存。
