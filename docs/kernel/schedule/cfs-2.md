# load_weight与vruntime

## 前言

调度类`fair_sched_class`中规定了调度器的基本操作集合，`cfs_rq`实现了被操作的就绪队列。剩下的就是研究操作集合中的具体实现，看看CFS是如何管理`cfs_rq`中的进程的。本文主要解释了两个问题：

- 什么样的任务归CFS管
- CFS如何实现队列内部的优先级划分

## 调度策略与调度类

| sched class |               sched policy                |
| :---------: | :---------------------------------------: |
|    fair     | SCHED_NORMAL or SCHED_BATCH or SCHED_IDLE |
|     rt      |          SCHED_FIFO or SCHED_RR           |
|     dl      |              SCHED_DEADLINE               |

每一种调度类实现了一类调度策略。当设定任务的调度策略为`SCHED_NORMAL`或者`SCHED_BATCH`时该任务会被放入CFS的就绪队列中。这里有一个点需要注意，当设定的调度策略为`SCHED_IDLE`时，该任务实际放入的是CFS的就绪队列，与`idle_sched_class`没有关系。只是其权重为`3`，比CFS中最低权重`15`还要小。

## 任务优先级&权重&虚拟时间

```c
struct sched_entity {
    struct load_weight		load;               // 权重信息
}
```

在内核中，每一个Task都有一个调度实体`sched_entity`记录其调度信息。其中`struct load_weight`记录着的权重信息.

在CFS中`vruntime`最小的Task会被优先调度，在引入了优先级以后我们希望高优先级的任务能够得到更多的运行时间，CFS利用权重值影响`vruntime`的计算，每一次调度时将Wall-Time转化为`vruntime`并累计到对应的`sched_entity`上。不同优先级的Task运行相同的Wall-Time的情况下，高优先级的Task具备高权重，得到的`vruntime`较小，低优先级的Task具备低权重得到的`vruntime`较高，这样一段时间运行下来，高优先级的Task会被调度的次数更多，运行的时间更长。

```c
#define WEIGHT_IDLEPRIO		3              
#define WMULT_IDLEPRIO		1431655765
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};

const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```

在Linux中，CFS使用Nice值来表示优先级，Nice值越小其权重越大，可以理解为Nice值越小优先级越高，Nice值范围为`[-20,19]`。通过查表`sched_prio_to_weight`可以实现Nice值到权重值的转化。任务的默认Nice值为0，对应的权重为1024，任意Nice值的权重通过`1024*1.25^(-nice)`计算。比如Nice值为-1，其权重值计算为`1024*1.25=1280`，这里会发现与表中的`1277`存在偏差，这是因为Linux内核在取值上进行了微调，具体微调的原因与`vruntime`的计算函数有关。

`sched_prio_to_wmult`是权重值表的反转值表，这个表由`sched_prio_to_weight`计算的得来，其计算方式为`sched_prio_to_wmult[prio] = (2^32) / sched_prio_to_weight[prio]`。这个表存在的目的是通过预计算的方式将除法运算转化为乘法运算，提高计算速度。比如Nice值为0的任务计算为`sched_prio_to_wmult[20]=(2^32) / 1024 = 4194304`。

**NOTE:** `SCHED_IDLE`任务在CFS中使用`WEIGHT_IDLEPRIO` & `WMULT_IDLEPRIO` 设置权重。

```c
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
    if (unlikely(se->load.weight != NICE_0_LOAD))
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

    return delta;
}
```

接下来就需要研究一下三个问题，Wall-Time是如何计算出`vruntime`的？为什么权重值的设定是这样的？`sched_prio_to_wmult`这个表又是如何加速计算的？这三个问题都与计算的过程相关。
`calc_delta_fair()`根据`se`的权重信息计算Wall-Time对应的`vrtuntime`，当调度实体的权重等于`NICE_0_LOAD`时计算得到的`vruntime`与Wall-Time是一致，此时不需要转换直接返回。计算的关键部分在`__calc_delta()`中。

```c
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
    u64 fact = scale_load_down(weight);
    int shift = WMULT_SHIFT;
    __update_inv_weight(lw);
    if (unlikely(fact >> 32)) {
        while (fact >> 32) {
            fact >>= 1;
            shift--;
        }
    }
    /* hint to use a 32x32->64 mul */
    fact = (u64)(u32)fact * lw->inv_weight;
    while (fact >> 32) {
        fact >>= 1;
        shift--;
    }
    return mul_u64_u32_shr(delta_exec, fact, shift);
}
```

`__calc_delta()`函数接受三个参数，`delta_exec`是Wall-Time，`weight`是`NICE_0_LOAD`(Nice值0对应的权重值)，`lw`是任务的权重信息。这段代码比较长，其实可以不用细看，其计算了`vruntime = delta_exec * weight / lw->weight`。但是直接计算会用到除法，为了加速这里将计算公式进行一次转化`vruntime = (delta_exec * weight * 2^32) / lw->weight) >> 32`，公式里的`2^32 / lw->weight`可以替换为反转值`lw->inv_weight`得到`vruntime = (delta_exec * weight * lw->inv_weight) >> 32`，`weight`是常数`1024`。这里通过预计算出来的`lw->inv_weight`将除法转化为乘法来加速运算。至此解释了“Wall-Time是如何计算出`vruntime`的？”以及“`sched_prio_to_wmult`这个表又是如何加速计算的？”这两个问题。

关于为什么权重值为什么要以系数`1.25`计算，参考`core.c`中的说明。作者期望每提升一级优先级其相对的可运行时间增加`10%`，其他任务的可运行时间减少`10%`。这里需要理解相对的含义。首先依据上述计算公式的逆运算可以计算`vruntime`对应的Wall-Time: `delta_exec = vruntime * lw->weight >> 32 / 1024`，在这个公式中`1024`和`vruntime`都是常数（因为CFS的目标就是`vruntime`相等），因此计算Wall-Time的占比就是计算其权重的占比。举一个例子，假设此时有三个任务权重分别为820、1024、1024，其Wall-Time占比分别为：`820/2868=28.5%`、`1024/2868=35.7%`、`35.7%`。此时将任务3的权重提高一级到`1277`，此时Wall-Time的占比分别为`26.2%`、`32.8%`、`40.9%`。此时可以看到任务3的占比上升`(40.9% - 35.7%) = 5.2%`，另外两个任务共同下降`(35.7% - 32.8%) + (28.5% - 26.2%) = 2.9% + 2.3% = 5.2%`，相对增加了`10.4%`。`10%`这个相对比例与队列中任务的数量及其权重应该是无关的，是算法层面能保证得到这个近似值。
