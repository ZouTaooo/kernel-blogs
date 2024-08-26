# 调度周期与最小抢占粒度

## 前言

CFS调度器的工作原理是根据任务的权重公平地分割物理时间，在每个调度周期内给予每个任务一定的执行机会。虽然任务的实际运行时间可能各不相同，但理论上，队列内的所有任务都会在周期内得到执行的机会。

**2023.11.21更新** ：之前对于调度周期的理解还不够全面，调度周期在真实的Linux内核中并不能保证队列中的每一个任务都能够有机会执行，触发调度的情况有任务主动让出执行权和时钟Tick中断两种，在`100HZ`的机器上每一次Tick的间隔为`10ms`，忽略其他会产生调度的情况，如果队列中有`10`个任务都跑满了各自的时间片，则每个任务都运行一次的周期是`100ms`，而以文中四核的例子计算的调度周期应该是`22.5ms`，这说明在低HZ机器上真实的调度周期和理论计算值间的误差会很大，即使在高HZ机器上也只能降低了误差，因此调度周期的准确性还取决于Tick的频率。

## 调度周期

```c
static u64 __sched_period(unsigned long nr_running)
{
	if (unlikely(nr_running > sched_nr_latency))
		return nr_running * sysctl_sched_min_granularity;
	else
		return sysctl_sched_latency;
}
```

调度周期的计算函数`__sched_period()`的计算结果与以下参数有关：

- `nr_running`: 当前`cfs_rq`中的Task数量
- `sysctl_sched_min_granularity`: 可配置参数，最小抢占粒度，任务被抢占前的最小运行时间。默认值为`0.75 msec * (1 + ilog(ncpus))`。
- `sysctl_sched_latency`: 可配置参数，调度延迟保证。默认值为`6ms * (1 + ilog(ncpus)）`。
- `sched_nr_latency`: 等于`sysctl_sched_latency / sysctl_sched_min_granularity`

```bash
$cat /proc/sys/kernel/sched_latency_ns
18000000
$cat /proc/sys/kernel/sched_min_granularity_ns
2250000
```

在一个四核的机器，其默认的`sched_latency_ns`为18ms，默认的`sched_min_granularity_ns`为2.25ms。`sched_nr_latency`自然为8。因此，当`cfs_rq`的任务数量小于等于8个时，此时的调度周期为`sched_latency_ns`，保证在18ms内每一个任务能够调度一次。如果任务超过了8个，比如当前`cfs_rq`有100个Task，调度周期为225ms，只能保证在225ms内每一个任务至少执行一次。按照抢占粒度进行调整是为了避免过小的调度周期导致时间片用完的上下文切换发生过于频繁，影响系统整体吞吐量。

## tick抢占检查

Tick抢占检查的由`check_preempt_tick()`实现：

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

当时钟Tick进行抢占检查时会判断以下两个条件，满足其中之一才符合抢占条件：

- 条件一: `curr`的实际运行物理时间超过了这个调度周期内理想的分配时间
- 条件二: 实际的物理运行时间少于分配的时间，但是超过了最小抢占粒度，并且`curr`与当前`vruntime`最小的任务之间的虚拟时间差值`delta`超过了`curr`的理想运行物理时间

条件一合乎常理，超过了应该运行的时间上限当然应该让出CPU。

条件二限定了抢占一个运行时间还不满足其分配的时间的任务的必须条件，首先是`curr`的运行时间不能太短，必须保证最小抢占粒度，避免抢占过于频繁。其次要求`curr`与当前`vruntime`最小的任务之间的虚拟时间差值`delta`超过了`curr`的理想运行物理时间，这有两个目的。第一个目的是如果当前`vruntime`最小的`se`和`curr`之间的差值过大时能够尽快执行（比如为了保障某些被唤醒的`se`可以尽快执行）。第二个目的是让高优先级的任务能够更容易抢占低优先级任务，这里用到了一种`trick`，用虚拟时间的差值`delta`与物理时间`ideal_runtime`比较，假设`se`的优先级高于`curr`，`delta`值高于`ideal_runtime`的概率就会更高一些，也就更容易抢占`curr`。
