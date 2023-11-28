<!-- CFS（三）调度周期 -->
## 前言

CFS的调度逻辑是将一个调度周期的物理时间按照任务的权重分配给每一个任务，确保在一个调度周期内，队列中的每一个任务都能够得到执行，尽管执行的时长会有差异。本文对调度周期的设定规则以及配置参数进行探究。

**2023.11.21更新** ：之前对于调度周期的理解还不够全面，调度周期在真实的Linux内核中并不能保证队列中的每一个任务都能够有机会执行，真实的调度除了主动让出执行权外主要由时钟tick驱动，在`100HZ`的机器上每一次tick的间隔在`10ms`，忽略其他会产生调度的情况，如果队列中有10个任务都跑满了，则每个任务都运行一次的周期是`100ms`，以文中四核的例子计算调度周期应该是`22.5ms`，因此在低HZ上真实的调度周期和计算值间的误差会很大，即使在高HZ核上也只能是降低了误差，因此调度周期准不准还得看CPU的HZ高不高。

## 调度周期


调度周期的计算函数`__sched_period`如下，计算结果与以下参数有关：
- `nr_running`: 当前`cfs_rq`中的task数量
- `sysctl_sched_min_granularity`: 可配置参数，最小抢占粒度，任务被抢占前的最小运行时间。默认为`0.75 msec * (1 + ilog(ncpus))`。
- `sysctl_sched_latency`: 可配置参数，调度延迟保证。默认为`6ms * (1 + ilog(ncpus)）`。
- `sched_nr_latency`: 始终等于`sysctl_sched_latency / sysctl_sched_min_granularity`
  
```c
static u64 __sched_period(unsigned long nr_running)
{
	if (unlikely(nr_running > sched_nr_latency))
		return nr_running * sysctl_sched_min_granularity;
	else
		return sysctl_sched_latency;
}
```

假设一个四核的场景，其默认的`sched_latency_ns`为18ms，默认的`sched_min_granularity_ns`为2.25ms。`sched_nr_latency`自然为8。

```bash
$cat /proc/sys/kernel/sched_latency_ns
18000000
$cat /proc/sys/kernel/sched_min_granularity_ns
2250000
```

因此，当`cfs_rq`的任务小于等于8个时，此时的调度周期为`sched_latency_ns`，保证在18ms内每一个任务能够调度一次。如果任务超过了8个，比如当前`cfs_rq`有100个task，调度周期为225ms，只能保证在225ms内每一个任务至少执行一次。这是因为如果不按照抢占粒度进行调整，过小的调度周期会导致由于时间片用完的抢占频繁出现影响整体吞吐。

## tick抢占检查

当时钟tick检查是否可以抢占当前任务时需要满足以下两个条件之一：
- 条件一: `curr`的实际运行物理时间超过了这个调度周期内理想的分配时间
- 条件二: 实际的物理运行时间少于分配的时间，但是超过了最小抢占粒度，并且`curr`与当前`vruntime`最小的任务之间的虚拟时间差值`delta`超过了`curr`的理想运行物理时间

条件一合乎常理，超过了应该运行的时间当然应该让出CPU。条件二限定了抢占一个运行时间还不满足其分配的时间的任务的必须条件，首先是`curr`的运行时间不能太短，必须保证最小抢占粒度，避免抢占过于频繁；如果`curr`满足最小的抢占粒度，要让高优先级的任务更容易抢占低优先级的任务，这里用到了一种trick，用虚拟时间的差值`delta`与物理时间`ideal_runtime`比较。假设`se`的优先级高于`curr`，`delta`值高于`ideal_runtime`的概率就会更高一些，也就更容易抢占`curr`。另一个方面是，是如果`curr`是一个低优先级任务其`ideal_runtime`会更小，更容易被强占。结合上述的虚拟时间与物理时间的比较，高优先级任务从两个方面都提高了抢占的成功概率。

```c
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
	ideal_runtime = sched_slice(cfs_rq, curr);
	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    /* 实际运行时间超过一个调度周期内的分配时间 */
	if (delta_exec > ideal_runtime) {
		resched_curr(rq_of(cfs_rq));
		return;
	}
    /* 不足分配的时间的前提下，不满足最小运行粒度 */
	if (delta_exec < sysctl_sched_min_granularity)
		return;
    
    /* 虚拟时间之差超过当前任务的理想运行物理时间 */
	se = __pick_first_entity(cfs_rq);
	delta = curr->vruntime - se->vruntime;

	if (delta < 0)
		return;

	if (delta > ideal_runtime)
		resched_curr(rq_of(cfs_rq));
}
```

## 总结

* CFS中的调度周期的概念，调度周期保证了这个时间内每一个任务至少执行一次。
* 调度周期的影响因素有`sysctl_sched_latency`以及`nr_runnning & sysctl_sched_min_granularity`
* tick抢占会考虑到`sysctl_sched_min_granularity`，避免频繁出现`tick`抢占。
