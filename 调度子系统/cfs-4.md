<!-- CFS 新任务task创建 -->

## 前言

新任务产生接口有`clone`、`fork`等系统调用，这些系统调用的都是通过`do_fork`函数实现。本文主要对`do_fork`中CFS的调度信息是如何初始化的进行了探究。

## CFS的调度信息初始化

```c
long _do_fork(...)
{
    p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
    wake_up_new_task(p);
}
```
在`_do_fork`中大量的初始化操作在`copy_process`中完成，其中和调度初始化有关的由`sched_fork`完成。

```c
int sched_fork(unsigned long clone_flags, struct task_struct *p)
{
    /* reset调度数据结构 */
	__sched_fork(clone_flags, p);
	
	p->state = TASK_NEW;
    /* 确保优先级不会随父进程突增 */
	p->prio = current->normal_prio;
    /* sched_reset_on_fork reset优先级信息 */
    if (unlikely(p->sched_reset_on_fork)) {
		if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
			p->policy = SCHED_NORMAL;
			p->static_prio = NICE_TO_PRIO(0);
			p->rt_priority = 0;
		} else if (PRIO_TO_NICE(p->static_prio) < 0)
			p->static_prio = NICE_TO_PRIO(0);

		p->prio = p->normal_prio = __normal_prio(p);
		set_load_weight(p, false);
    	p->sched_reset_on_fork = 0;
	}
    /* 设置调度类 */
	if (dl_prio(p->prio))
		return -EAGAIN;
	else if (rt_prio(p->prio))
		p->sched_class = &rt_sched_class;
	else
		p->sched_class = &fair_sched_class;
    /* 调用CFS的task_fork_fair */
	if (p->sched_class->task_fork)
		p->sched_class->task_fork(p);
}
```
`sched_fork`函数在`core.c`中，在最后一行之前都是调度层面的初始化，
- `__sched_fork`将任务的CFS相关数据清零
- 设置`p->state`为`TASK_NEW`表示task还在初始化过程中避免被调度
- 将`prio`设置为`current->normal_prio`是一个比较关键的点，在Linux中`prio`是调度决策时使用的优先级，新任务的优先级默认会继承父进程的优先级，但是继承的必须是`normal_prio`，因为`prio`在特殊情况下会出现临时提高的情况，有可能不是任务正常的优先级。
- `sched_reset_on_fork`标志位的处理，设置此标志位时新任务需要重置优先级为`120`（`120`是`nice=0`的CFS任务对应的优先级），调度策略`policy`为`SCHED_NORMAL`。`set_load_weight`会更新任务的权重信息。如果没有该标志位，自然使用的是父进程的优先级信息。
- 根据`prio`识别和设置所属调度类
- 关键一步，如果是`fair_sched_class`会调用调度类的`task_fork`函数，在CFS中的对应实现为`task_fork_fair`

截止目前为止，`task`的调度策略、优先级、权重信息（如果是CFS）都已经通过拷贝父进程或者重置的方式初始化完成。但是作为一个`normal-task`CFS调度所依赖的`vruntime`还未初始化。`task_fork_fair`会对新任务的`vruntime`进行初始化。
```c
static void task_fork_fair(struct task_struct *p)
{
    /* 更新rq时钟 */
	update_rq_clock(rq);
    /* 如果是curr调用的fork的处理 */
	curr = cfs_rq->curr;
	if (curr) {
		update_curr(cfs_rq);
		se->vruntime = curr->vruntime;
	}
    /* 设置新任务的vruntime */
	place_entity(cfs_rq, se, 1);
    /* 可能出现的反转 */
	if (sysctl_sched_child_runs_first && curr && entity_before(curr, se)) {
		swap(curr->vruntime, se->vruntime);
		resched_curr(rq);
	}
    /* 得到相对值 */
	se->vruntime -= cfs_rq->min_vruntime;
}
```
主要流程和内容如下：
- 更新rq时钟信息，后续如果需要更新`curr`的`vruntime`需要用到
- 检查`curr`是否存在，`curr`是当前CFS队列中正在运行task，如果`curr`存在说明是`curr`调用的`fork`在创建新任务，此时需要先更新`curr`的`vruntime`再让子任务继承。`update_curr`核心就做两件事，更新`curr`的`vruntime`，更新`cfs_rq`的`min_vruntime`。如果`curr`不存在说明`fork`的调用者不归`fair_sched_class`管辖，不涉及到`vruntime`的更新。
```c
static void update_curr(struct cfs_rq *cfs_rq)
{
	curr->vruntime += calc_delta_fair(delta_exec, curr); /* wall-time转化为虚拟时间 */
	update_min_vruntime(cfs_rq); /* 更新min_vruntime */
}
```
- 关键一步`place_entity`设置新任务的`vruntime`。`place_entity`会惩罚新创建的task，因为当前的调度周期已经分配给任务队列中的任务了，如果直接插入会影响其他的任务。奖励sleep归来的task。
```c
static void
place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int initial)
{
	u64 vruntime = cfs_rq->min_vruntime;

	if (initial && sched_feat(START_DEBIT))
		vruntime += sched_vslice(cfs_rq, se);

	if (!initial) {
		unsigned long thresh = sysctl_sched_latency;

		if (sched_feat(GENTLE_FAIR_SLEEPERS))
			thresh >>= 1;

		vruntime -= thresh;
	}

	se->vruntime = max_vruntime(se->vruntime, vruntime);
}
```