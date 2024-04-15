<!-- CFS（六）PELT负载统计 -->

## 前言

`PELT`全称`per-entity load tracking`，用于实现调度实体级别的负载信息统计，能够为调度决策提供更细粒度的信息。上文中的组调度的任务组权重分配就依赖于负载信息，除此之外负载均衡场景也需要精准的对每个核的负载情况进行分析，`PELT`相比于`rq`级的负载统计，除了能知道负载的情况还能够知道每个调度实体负载的贡献度。

**Note**：`sched_avg->load_avg`与`uptime`和`top`命令中的`load1 load5 load15`等负载是无关的，`PELT`的负载仅用于系统中的调度决策。

## 相关数据结构

`sched_avg`是记录负载信息的数据结构，最终的计算结果包括`load_avg`，`runnable_load_avg`以及`util_avg`。在`sched_entity`和`cfs_rq`中都含有一个`sched_avg`用以记录负载信息，但是某些变量具备不同的含义。
* `load_avg`: 平均负载，对于`se`来说，`load_avg`与处于`runnable`状态的时间占比有关。对于`cfs_rq`来说`load_avg`由所有`runnable`和`blocked`状态的`se`的负载聚合得到。
* `runnable_load_avg`:对`se`来说和`load_avg`一致。对`cfs_rq`来说`runnable_load_avg`只考虑`runnable`状态的`se`，也就是`se`出队时会减去相应的负载。
* `util_avg`: `running`状态的负载情况，与运行在CPU上的时间有关。


```c
struct sched_avg {
    u64				last_update_time;            /* 最后更新时间 */
    u64				load_sum;                    /* 负载总和(runnable+blocked) */
    u64				runnable_load_sum;           /* 可运行任务的负载 */
    u32				util_sum;                    /* running任务的负载 */
    u32				period_contrib;              /* 计算负载时不足1024us的部分（d3） */
    unsigned long			load_avg;            /* 平均负载(runnable+blocked) */
    unsigned long			runnable_load_avg;   /* 可运行任务的平均负载（runnable） */
    unsigned long			util_avg;            /* 平均利用率 */
} ____cacheline_aligned;

struct sched_entity {
    struct load_weight		load;
    unsigned long			runnable_weight;
    struct sched_avg		avg;
}

struct cfs_rq {
    struct load_weight	load;
    unsigned long		runnable_weight;
    struct sched_avg		avg;
}
```

## 负载的计算

负载的计算目标是得到`load_avg`，`runnable_load_avg`以及`util_avg`。`__update_load_avg_se`会对调度实体`se`的负载进行更新，调度实体分为两种`group-se`和`task-se`，`group-se`用于支持组调度。

```c
int __update_load_avg_se(u64 now, int cpu, struct cfs_rq *cfs_rq, struct sched_entity *se)
{   
    /* 1. task-se runnable_weight load_weight相同 */
    if (entity_is_task(se))
        se->runnable_weight = se->load.weight;
    /* 2. 更新 *_sum */
    if (___update_load_sum(now, cpu, &se->avg, !!se->on_rq, !!se->on_rq,
                cfs_rq->curr == se)) {
        /* 3.更新 *_avg */
        ___update_load_avg(&se->avg, se_weight(se), se_runnable(se));
        cfs_se_util_change(&se->avg);
        return 1;
    }

    return 0;
}
```
负载更新主要有两步，`___update_load_sum`更新`*_sum`，`___update_load_avg`更新`*_avg`得到平均负载等指标。

### 算法分析

在看源码前先了解一下大概的算法，假设当前为`now`，设计者认为距离`now`越久远对应的负载的贡献度就越低，为了实现这个想法，算法将从开始执行`start`到`now`按照`1024us`（大约`1ms`）划分为多个段，如下所示。这样可以计算每个时间段内的负载贡献。
```c
 |<- 1024us ->|<- 1024us ->|<- 1024us ->| ...
now   p0    prev    p1           p2
     (now)       (~1ms ago)  (~2ms ago)
```
随着时间流逝，距离`now`越久远的负载其贡献度要乘上一个更小的系数，让其不断地下降，最简单的做法就是记录历史的每个时间段的负载信息，但是这样会有很大的开销。设计者采取了一个很巧妙的方法，通过设定一个系数`y^32 = 0.5 y~=0.97857206208`，如上图假设每个时间段的负载依次为`u_0,u_1,...u_n`，`now`时刻的负载计算方法为`load(now) = u_0 + u_1*y + u_2*y^2 + u_3*y^3 + ...`，通过归纳`load(n) = load(n-1) * y`，这样就无需记录历史的每个周期的负载信息，同时保证了随着时间流逝距离越远的负载的贡献度越低，每32个周期过去，该时间段的负载贡献度下降一半。

上面是算法的理想状态，实际上内核并不是每`1024us`周期时触发一次负载更新，因此有可能会存在计算时有不足`1024us`的部分，如果上次计算时存在不足`1024us`，本次计算还需考虑补全上次不足的部分。如下所示，假设`now`距离`prev`计算负载时过了`2872us`，此时被分为三个部分`d1(200us) d2(2048us) d3(624us)`。
```c
|<-924us->|<-200us->|<-----1024us------>|<-----1024us----->|<--624us->|<-500us->|
         prev                                                        now

            d1          d2           d3
            ^           ^            ^
            |           |            |
            |<->|<----------------->|<--->|
    ... |---x---|------| ... |------|-----x (now)
```
新增负载的**计算公式(1)** 如下，`y`是上面的系数`0.97857...`，`p`是跨越的周期数在这就是3。
```c
                     p-1
      d1 y^p + 1024 \Sum y^n + d3 y^0		(1)
                     n=1
```

因此在这个例子下计算当前负载需要将原始负载做衰减加上新增的负载部分`load(now) = load(prev) * y^3 + d1 * y^3 + 1024 * (y^2 + y) + d3`。从公式可以看出计算贡献度的关键是找出跨越的周期数`p`以及`d1`和`d3`的值，`d2`部分可以通过等比数列求和来计算。


按照上述公式的如果总是这样累加下去，最终得到的只是一个等比数列求和的极限值，因此要去除没有负载贡献的部分。设计者采取是采样的方法，在每次更新做一次状态判断，决定是否将当前时刻的贡献度累加到负载上，比如`se`的`load_avg`只有在当前`se`处于`on_rq`状态时才做累计，如果此时`se`更新时并不在`cfs_rq`中这段时间的负载贡献度就被忽略了，而历史的负载却要衰减，就会变小。

**Note**：这里我产生了一个疑问，在cpu上运行的调度实体`se->on_rq`是`1`还是`0`呢？这关系到负载的计算，通过查看源码，`on_rq`的含义应该是所有在`cfs_rq`中的`se`以及运行在cpu上的`se`，但是运行在cpu上的`se`在运行前是被移出了`cfs_rq`的，在运行结束后，选择好下一个任务以后再移入`cfs_rq`。

回到源码上，`__accumulate_pelt_segments()`实现了上述 **公式(1)** 的算法，`__accumulate_pelt_segments()`在计算上有两处优化:
* 第一处是求`val*y^n`，代码上由`decay_load`实现，由于`y^n`次方是固定的，因此内核提前做了打表操作。如果n超过了表的大小，需要做一次转化`y^n = (1/2)^(n/32) * val^(n%32)`，这样可以`n%32`就可以查表了。这里的`32`对应着宏`LOAD_AVG_PERIOD`.
* 第二处是求中间的等比数列的优化，利用到了极限求和，数学公式也比较简单，请自行阅读注释部分。

```c
static u32 __accumulate_pelt_segments(u64 periods, u32 d1, u32 d3)
{
    u32 c1, c2, c3 = d3; /* y^0 == 1 */
    /*
     * c1 = d1 y^p
     */
    c1 = decay_load((u64)d1, periods);
    /*
     *            p-1
     * c2 = 1024 \Sum y^n
     *            n=1
     *
     *              inf        inf
     *    = 1024 ( \Sum y^n - \Sum y^n - y^0 )
     *              n=0        n=p
     */
    c2 = LOAD_AVG_MAX - decay_load(LOAD_AVG_MAX, periods) - 1024;

    return c1 + c2 + c3;
}
/* 提前打好的表 */
static const u32 runnable_avg_yN_inv[] = {
    0xffffffff, 0xfa83b2da, 0xf5257d14, 0xefe4b99a, 0xeac0c6e6, 0xe5b906e6,
    0xe0ccdeeb, 0xdbfbb796, 0xd744fcc9, 0xd2a81d91, 0xce248c14, 0xc9b9bd85,
    0xc5672a10, 0xc12c4cc9, 0xbd08a39e, 0xb8fbaf46, 0xb504f333, 0xb123f581,
    0xad583ee9, 0xa9a15ab4, 0xa5fed6a9, 0xa2704302, 0x9ef5325f, 0x9b8d39b9,
    0x9837f050, 0x94f4efa8, 0x91c3d373, 0x8ea4398a, 0x8b95c1e3, 0x88980e80,
    0x85aac367, 0x82cd8698,
};
/* 衰减1/2的周期 */
#define LOAD_AVG_PERIOD 32
/* 平均负载的最大值 */
#define LOAD_AVG_MAX 47742
```

### 源码分析

#### ___update_load_sum计算*_sum
这里先关注第一步`___update_load_sum()`如何计算`*_sum`这个中间数据：
* 为了避免频繁更新`___update_load_sum()`设置了至少`1us`的更新间隔
* 更新前设置`sched_avg.last_update_time`用于下次更新时计算时间间隔。
* 检查`load` `runnable` `running`三个参数，分别表示是本周期计算的贡献度是否累计到对应的负载上。如果不贡献给`load_sum`也就不需要贡献给`runnable_load_sum`和`util_sum`。
* 调用`accumulate_sum`计算当前周期的贡献度，会用到上述的算法部分。
```c
static __always_inline int
___update_load_sum(u64 now, int cpu, struct sched_avg *sa,
          unsigned long load, unsigned long runnable, int running)
{
    u64 delta;
    /* 1. 不足1us不计算 */
    delta = now - sa->last_update_time;
    delta >>= 10;
    if (!delta)
        return 0;
    
    /* 2. 记录上一次的更新时间 */
    sa->last_update_time += delta << 10;
    /* 3. 如果不对load_sum做贡献 也不需要对runnable_load_sum 和 util_sum做贡献了 */
    if (!load)
        runnable = running = 0;
    /* 4. 将历史负载做衰退、计算这个新周期的负载贡献度、判断是否需要累积到对应的负载上 */
    if (!accumulate_sum(delta, cpu, sa, load, runnable, running))
        return 0;

    return 1;
}
```

重点关注`accumulate_sum()`，在上述的算法部分我们只考虑了时间的维度，但是真实的负载是与`CPU`的性能相关的。一个多核系统中核与核的体质不能一概而论，不同频率的`CPU`共存于同一系统是可能的，因此为了让不同核上任务的负载有可对比性需要对CPU的性能做归一化处理，相同的运行时间在大核上理应得到更高的负载贡献，`scale_freq`和`scale_cpu`(核的算力存在`freq`和`capacity`的差异，算力评估需要得到频率一致性和容量一致性，频率好理解，`capacity`指的是单位时间内执行的指令数量)就是归一化后`CPU`性能相关参数，范围均处于`[0,1024]`（最大核的参数对应`1024`，小核按比例折算）。

回到代码上，在进入计算前需要计算实际上跨越的周期个数`periods`，此时有三种情况：
* `periods==0`:只需要考虑`d1`部分
* `periods==1`:此时只有`d1`和`d3`两个部分
* `periods>=2`:此时会存在`d1` `d2` `d3`三个部分。

计算`d1 = 1024 - sa->period_contrib` `d3 = delta % 1024`，`periods` `d1` `d3`三个参数作为输入调用`__accumulate_pelt_segments`按照**计算公式(1)** 求出`contrib`。然后就是更新各个负载`*_sum`了，在这里`contrib`需要乘上所在核的频率系数，`load` `runnable` `running`在`__update_load_avg_se`中都被调整为`0`或`1`，用以实现采样式的更新。

```c
static __always_inline u32
accumulate_sum(u64 delta, int cpu, struct sched_avg *sa,
           unsigned long load, unsigned long runnable, int running)
{
    unsigned long scale_freq, scale_cpu;
    /* p == 0 -> delta < 1024 情况下 */
    u32 contrib = (u32)delta;
    u64 periods;
    /* 获取归一化的CPU性能参数 */
    scale_freq = arch_scale_freq_capacity(cpu);
    scale_cpu = arch_scale_cpu_capacity(NULL, cpu);
    /* 1. 计算跨过的周期数 */
    delta += sa->period_contrib;
    periods = delta / 1024; /* A period is 1024us (~1ms) */
    
    if (periods) {
        /* 2. 衰减*_sum */
        sa->load_sum = decay_load(sa->load_sum, periods);
        sa->runnable_load_sum =
            decay_load(sa->runnable_load_sum, periods);
        sa->util_sum = decay_load((u64)(sa->util_sum), periods);
        /*  d1 = 1024 - sa->period_contrib
            d3 = delta % 1024 */
        delta %= 1024;
        /* 3. 计算贡献度 */
        contrib = __accumulate_pelt_segments(periods,
                1024 - sa->period_contrib, delta);
    }
    /* 更新已贡献周期，下次更新时可以方便计算d1 */
    sa->period_contrib = delta;
    /* 将scale_freq归一化到 [0,1]， contrib = [0,1] * contrib */
    contrib = cap_scale(contrib, scale_freq);
    /* 更新 *_sum */
    if (load)
        sa->load_sum += load * contrib;
    if (runnable)
        sa->runnable_load_sum += runnable * contrib;
    if (running)
        sa->util_sum += contrib * scale_cpu;

    return periods;
}
```


#### ___update_load_avg计算*_avg

`*_sum`通过时间与CPU频率系数的计算表示出了算力负载的概念，而平均负载需要除去过去总的时间，需要注意历史负载的贡献度在随着时间衰退，计算平均值时每个时间段的时长也要进行相应的衰退才能得到每个`1024us`时间段内的平均负载。求分母的公式如下，又要扯到一点数学。

```bash
|<- sa->period_contrib ->|<- 1024us ->|<- 1024us ->|<- 1024us ->| ...

                   inf
divider = 1024 * ( \Sum y^n ) + sa->period_contrib
                   n=1
                   inf
        = 1024 * ( \Sum y^n - y^0 ) + sa->period_contrib
                   n=0
        = LOAD_AVG_MAX - 1024 + sa->preiod_contrib           
```

回到代码上，这里的`load`与`runnable`分别是负载权重`load.weight`以及`runnable_weight`。从理论上讲，如果一个任务始终处于运行状态，根据负载的计算方法`sa->load_sum`会不断逼近`LOAD_AVG_MAX`，所以`sa->load_avg`也会不断逼近于`load.weight`（应该要假设所有的核的频率一致）。可以看到在这里负载还将调度实体的权重值也放入考虑范围，`CFS`认为高权重的任务具备更高的负载贡献度，这也是很合理的，假如有两个核，一个核上全部跑着高权重的任务，另一个核上全部跑低权重的任务，即使两个核的`CPU`利用率都跑满了也不能认为两个核的负载是相同的，只有高权重任务和低权重任务在两个核上的分布均匀时才能认为负载是差不多的，这也是负载均衡要解决的问题。

```c
static __always_inline void
___update_load_avg(struct sched_avg *sa, unsigned long load, unsigned long runnable)
{
    u32 divider = LOAD_AVG_MAX - 1024 + sa->period_contrib;

    sa->load_avg = div_u64(load * sa->load_sum, divider);
    sa->runnable_load_avg =	div_u64(runnable * sa->runnable_load_sum, divider);
    WRITE_ONCE(sa->util_avg, sa->util_sum / divider);
}
```

