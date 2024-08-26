# 内存分配-GFP_MASK

## GFP_MASK

GFP是`Get Free Page`的缩写，GFP_MASK是一系列内存分配的掩码，掩码主要有两个作用

- 指导伙伴系统在合适的位置分配内存块，确定Node、Zone和自由链表
- 如果分配过程遇到内存不足可以用于指导分配的行为，比如允许失败或者触发内存回收后再次申请。

### Zone限制相关的GFP_MASK

这些标志可以限制分配的内存块所属的Zone。

```c
#define ___GFP_DMA		0x01u
#define ___GFP_HIGHMEM		0x02u
#define ___GFP_DMA32		0x04u
#define ___GFP_MOVABLE		0x08u
```

这四个GFP_MASK指示了伙伴系统优先从哪个Zone上进行内存分配。内核提供了`gfp_zone()`函数可以将GFP_MASK转换为`zone_type`。

```c
#define GFP_ZONE_TABLE ( \
    (ZONE_NORMAL << 0 * GFP_ZONES_SHIFT)				       \
    | (OPT_ZONE_DMA << ___GFP_DMA * GFP_ZONES_SHIFT)		       \
    | (OPT_ZONE_HIGHMEM << ___GFP_HIGHMEM * GFP_ZONES_SHIFT)	       \
    | (OPT_ZONE_DMA32 << ___GFP_DMA32 * GFP_ZONES_SHIFT)		       \
    | (ZONE_NORMAL << ___GFP_MOVABLE * GFP_ZONES_SHIFT)		       \
    | (OPT_ZONE_DMA << (___GFP_MOVABLE | ___GFP_DMA) * GFP_ZONES_SHIFT)    \
    | (ZONE_MOVABLE << (___GFP_MOVABLE | ___GFP_HIGHMEM) * GFP_ZONES_SHIFT)\
    | (OPT_ZONE_DMA32 << (___GFP_MOVABLE | ___GFP_DMA32) * GFP_ZONES_SHIFT)\
)

static inline enum zone_type gfp_zone(gfp_t flags)
{
    enum zone_type z;
    int bit = (__force int) (flags & GFP_ZONEMASK);

    z = (GFP_ZONE_TABLE >> (bit * GFP_ZONES_SHIFT)) &
                     ((1 << GFP_ZONES_SHIFT) - 1);
    return z;
}
```

`ZONE_NORMAL`和`ZONE_MOVABLE`并没有显式的GFP_MASK对应。当同时指定`__GFP_HIGHMEM`和`__GFP_MOVABLE`时会返回`ZONE_MOVABLE`，表示从虚拟区域`ZONE_MOVABLE`中分配内存，该区域和内存碎片以及虚拟化中内存的热插拔相关，仅指定`___GFP_MOVABLE`时会返回`ZONE_NORMAL`。

根据返回的`zone_type`区域类型，内存分配API就能确定需要扫描的内存区域有哪些，内存区域按照稀有程度由低到高排序为`ZONE_MOVEABLE`、`ZONE_HIGHMEM`、`ZONE_NORMAL`、`ZONE_DMA32`和`ZONE_DMA`，可搜索的内存区域包含了`gfp_zone()`返回的区域类型以及其他稀有程度更低的区域，比如返回类型为`ZONE_NORMAL`时扫描的区域包括NORMAL、DMA32和DMA，返回类型为HIGHMEM时扫描的区域则包括HIGHMEM、NORMAL、DMA32、DMA，以此类推。这里的稀有程度指的是内存区域的重要性，作用越大、越是通用的内存区域要让给更需要的场景使用。当`gfp_zone()`返回`ZONE_MOVABLE`时情况比较特殊，此时会在特殊的虚拟内存区中进行内存分配。

### 页面移动性限制相关的GFP_MASK

这些标志用于在内存分配时对关于页面移动性、放置策略进行限制。

```c
#define __GFP_MOVABLE	((__force gfp_t)___GFP_MOVABLE)  /* ZONE_MOVABLE allowed */
#define __GFP_RECLAIMABLE ((__force gfp_t)___GFP_RECLAIMABLE)
#define __GFP_WRITE	((__force gfp_t)___GFP_WRITE)
#define __GFP_HARDWALL   ((__force gfp_t)___GFP_HARDWALL)
#define __GFP_THISNODE	((__force gfp_t)___GFP_THISNODE)
#define __GFP_ACCOUNT	((__force gfp_t)___GFP_ACCOUNT)
```

- `__GFP_MOVABLE`: 这个标志位表明分配的页面可以在内存规则过程中被迁移或者可以被回收。`__GFP_MOVABLE`在Zone限制中也起作用。
- `__GFP_RECLAIMABLE`: 这个标志表示分配的页面是可以移动的。当进行内存规整时，内核可能会迁移页面以优化内存布局并减少外部碎片。具有`__GFP_MOVABLE`标志的页面可以被内核安全地移动。
- `__GFP_RECLAIMABLE`: 这个标志在标记了了`SLAB_RECLAIM_ACCOUNT`的slab分配中使用，意味着分配页面后续可以通过shrinker机制进行释放。
- `__GFP_WRITE`: 这个标志表明调用者会对分配的页面写入数据并标记为脏页，内核会尽可能地将这些页面分散到不同的区域从而保持平衡。
- `__GFP_HARDWALL`: 这个标志强制执行`cpuset`的内存分配策略，这表明分配的页面必须来自`cpuset`相关的Node。
- `__GFP_THISNODE`: 这个标志要求内存分配必须从当前Node获取，如果该Node上没有足够的内存不会尝试从其他Node分配。
- `__GFP_ACCOUNT：`: 这个标志表示申请的内存受到`kmemcg`的统计，`kmemcg`与内存资源的细粒度控制和隔离有关。

### 水位限制相关的GFP_MASK

这些标志位控制分配过程访问保留内存的策略。

```c
#define __GFP_ATOMIC	((__force gfp_t)___GFP_ATOMIC)
#define __GFP_HIGH	((__force gfp_t)___GFP_HIGH)
#define __GFP_MEMALLOC	((__force gfp_t)___GFP_MEMALLOC)
#define __GFP_NOMEMALLOC ((__force gfp_t)___GFP_NOMEMALLOC)
```

- `__GFP_HIGH`: 这个标志表明请求者具有高优先级，此次内存分配会影响系统继续运行。例如，在创建用于清理页面的I/O上下文时可能会使用这个标志。
- `__GFP_ATOMIC`: 指出调用者无法执行回收操作或进入休眠状态，并且此次内存分配具有高优先级。不允许睡眠的中断处理程序就是一个典型的调用者，这个标志可以与`__GFP_HIGH`一起使用。
- `__GFP_MEMALLOC`: 这个标志位允许访问所有类型的内存，但是为了避免完全耗尽内存调用者需要保证很快会有更多的内存被释放，例如在进程退出或交换过程中。
- `__GFP_NOMEMALLOC`: 用于明确禁止访问紧急保留内存。如果同时设置了`__GFP_MEMALLOC`和`__GFP_NOMEMALLOC`，则`__GFP_NOMEMALLOC`优先。
  
### 回收限制相关的GPF_MASK

这些标志在允许睡眠（没有设置`GFP_NOWAIT`和`GFP_ATOMIC`）的内存分配过程中指导内存回收。

```c
#define __GFP_IO	((__force gfp_t)___GFP_IO)
#define __GFP_FS	((__force gfp_t)___GFP_FS)
#define __GFP_DIRECT_RECLAIM	((__force gfp_t)___GFP_DIRECT_RECLAIM) /* Caller can reclaim */
#define __GFP_KSWAPD_RECLAIM	((__force gfp_t)___GFP_KSWAPD_RECLAIM) /* kswapd can wake */
#define __GFP_RECLAIM ((__force gfp_t)(___GFP_DIRECT_RECLAIM|___GFP_KSWAPD_RECLAIM))
#define __GFP_RETRY_MAYFAIL	((__force gfp_t)___GFP_RETRY_MAYFAIL)
#define __GFP_NOFAIL	((__force gfp_t)___GFP_NOFAIL)
#define __GFP_NORETRY	((__force gfp_t)___GFP_NORETRY)
```

- `__GFP_IO`: 在内存回收中允许启动物理I/O操作。
- `__GFP_FS`: 可以调用底层文件系统。清除该标志可以避免在已经持有锁的情况下进入文件系统，从而预防死锁。
- `__GFP_DIRECT_RECLAIM`: 表明调用者可以进入直接回收模式。清除这个标志位可以在还有备选内存可用的时候避免回收延迟。
- `__GFP_KSWAPD_RECLAIM`: 当达到内存总量达到低水位时唤醒`kswapd`进行内存回收直到达到高水位。如果清除此标志可以在存在备用内存可用时避免干扰系统。
- `__GFP_RECLAIM`: 可以同时允许或禁止直接内存回收和`kswapd`内存回收。

内核将分配阶高于`PAGE_ALLOC_COSTLY_ORDER`分配请求称作作昂贵的请求。内核根据请求的分配阶采取不同的策略，对于那些非昂贵的请求是不允许失败，同时昂贵的分配请求要尽可能不干扰系统，不允许调用OOM killer。下面的三个标志位用于调整这些默认策略。

- `__GFP_NORETRY`: 仅尝试轻量级的直接内存回收，避免触发如OOM Killer。分配的调用者必须处理失败情况。
- `__GFP_RETRY_MAYFAIL`: 将多次重试内存分配，可以等待内存规整、换出释放的内存。同样会有一个重试次数，但是比`__GFP_NORETRY`大的多，只有可用内存很少时才会出现失败。该标志位不会直接触发OOM Killer，但是如果失败表明OOM Killer之后被触发的可能性很大，因此调用者应该释放一部分无用内存，保障系统稳定运行。
- `__GFP_NOFAIL`: 必须无限次重试内存分配，因为调用者无法处理失败。虽然可能会阻塞，但永远不会返回失败，应该谨慎使用此标志。

### 动作限制相关的GFP_MASK

```c
#define __GFP_NOWARN	((__force gfp_t)___GFP_NOWARN)
#define __GFP_COMP	((__force gfp_t)___GFP_COMP)
#define __GFP_ZERO	((__force gfp_t)___GFP_ZERO)
```

- `__GFP_NOWARN`: 这个标志会抑制内存分配失败的报告。当分配失败时内核会生成警告或错误信息，使用`__GFP_NOWARN`即使分配失败也不会显示任何警告。
- `__GFP_COMP`: 这个标志用于处理复合页（Compound Page）的元数据。复合页是一种将多个物理页面组合成一个更大的逻辑页面的技术，以支持大对象或大内存区域的分配。
- `__GFP_ZERO`: 如果分配成功，返回的页面会被清零。

## 常用GFP_MASK组合

内核提供了一系列的常用flag组合，适用于绝大多数内核内存分配场景。

```c
#define GFP_ATOMIC	(__GFP_HIGH|__GFP_ATOMIC|__GFP_KSWAPD_RECLAIM)
#define GFP_KERNEL	(__GFP_RECLAIM | __GFP_IO | __GFP_FS)
#define GFP_KERNEL_ACCOUNT (GFP_KERNEL | __GFP_ACCOUNT)
#define GFP_NOWAIT	(__GFP_KSWAPD_RECLAIM)
#define GFP_NOIO	(__GFP_RECLAIM)
#define GFP_NOFS	(__GFP_RECLAIM | __GFP_IO)
#define GFP_USER	(__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
#define GFP_DMA		__GFP_DMA
#define GFP_DMA32	__GFP_DMA32
#define GFP_HIGHUSER	(GFP_USER | __GFP_HIGHMEM)
#define GFP_HIGHUSER_MOVABLE	(GFP_HIGHUSER | __GFP_MOVABLE)
#define GFP_TRANSHUGE_LIGHT	((GFP_HIGHUSER_MOVABLE | __GFP_COMP | \
			 __GFP_NOMEMALLOC | __GFP_NOWARN) & ~__GFP_RECLAIM)
#define GFP_TRANSHUGE	(GFP_TRANSHUGE_LIGHT | __GFP_DIRECT_RECLAIM)
```
