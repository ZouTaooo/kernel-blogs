# sched_wakeup_granularity

## 前言

将一个处于睡眠状态或者新创建的进程加入就绪队列时会产生唤醒抢占检查，被唤醒的任务一般期望能够立刻执行，发生抢占能够满足被唤醒任务的实时性需求。CFS调度器的唤醒抢占能否成功会受到`sysctl_sched_wakeup_granularity`的影响，该参数能控制唤醒抢占发生的概率。频繁的抢占有可能会造成大的上下文切换开销，影响业务吞吐，因此`sysctl_sched_wakeup_granularity`是调度性能优化的一个重要参数。

## 唤醒抢占的触发条件

CFS调度类中的每一个任务都有一个虚拟时间`vruntime`，虚拟时间小的进程会优先被调度，被唤醒任务`se`抢占当前任务`curr`的只需要满足虚拟时间差`vdiff`超过唤醒抢占粒度`gran`，显然可知粒度越小发生抢占的概率越大。

```c
// fair.c
static int
wakeup_preempt_entity(struct sched_entity *curr, struct sched_entity *se)
{
	int ret;
	s64 gran, vdiff = curr->vruntime - se->vruntime;
    ...
	gran = wakeup_gran(se);
	if (vdiff > gran) // 超过唤醒抢占的粒度，抢占成功
		return 1;
	return 0;
}
```

## 抢占粒度的影响因素

唤醒抢占粒度的计算结果受两方面影响：

1. `sysctl_sched_wakeup_granularity`：该参数作为基底直接影响唤醒粒度的大小，可以改变系统整体的唤醒抢占发生概率
2. 被唤醒任务的权重：抢占唤醒还需要考虑到任务之间的优先级差异，调度器会优先保障高优先级任务的执行权，换句话说如果被唤醒任务的优先级高于正在执行的任务那么抢占就会更容易发生，反之抢占会更容易失败。

因此`calc_delta_fair()`将`sysctl_sched_wakeup_granularity`按照被唤醒任务`se`的权重进行了加工，`se`的权重越大计算得到的唤醒抢占粒度越小。

```c
// fair.c
static unsigned long wakeup_gran(struct sched_entity *se)
{
	unsigned long gran = sysctl_sched_wakeup_granularity;
    ... 
	return calc_delta_fair(gran, se);
}
```

## 调整sysctl_sched_wakeup_granularity

默认情况下`sysctl_sched_wakeup_granularity`为`1 msec * (1 + ilog(ncpus))`，比如4核机器默认值为3ms。

`Linux v5.13`之前的版本可以通过`echo X > /proc/sys/kernel/sched_wakeup_granularity`或者`sudo sysctl kernel.sched_wakeup_granularity_ns=X`进行修改。

`Linux v5.13`以后参数被移动到了`/sys/kernel/debug/sched/*`目录下。
