# Linux任务的优先级机制

## 前言

`task_struct`结构体中包含有四个优先级相关的成员变量，`prio`、`static_prio`、`normal_prio`和`rt_priority`这几个优先级值有什么区别和联系呢？

```c
struct task_struct {
    int				prio;
    int				static_prio;
    int				normal_prio;
    unsigned int       rt_priority;
}
```

## 内核中的优先级范围

```c
+----+--------------------------------------------------------+---------------+
| -1 |0                                                     99|100         139|   
+----+--------------------------------------------------------+---------------+
```

内核中管理多任务的调度类有三个，分别是`dl_sched_class`、`rt_sched_class`、`fair_sched_class`。其中`-1`优先级用于所有的Deadline-Task。`[0,99]`是Realtime-Task的优先级，`[100,139]`是Fair-Task的优先级。在内核优先值越低的任务优先级越高。

## static_prio

在内核中`static_prio`用于Fair-Task，这个值与Nice存在如下转化关系：`static_prio = nice + 120`。可以看到`static_prio`与`nice值`是绑定的。在一个任务的生命周期内`static_prio`变化的情况有两种：

* 系统调用`fork`: 在新任务初始化时，`sched_fork`会重置或者继承父进程的`static_prio`
* 系统调用`sys_setpriority`: 调用`set_user_nice()`更新进程的`static_prio`

系统调用`sys_setpriority()`可以设置用户、进程组和进程的优先级。当设置某个进程的Nice值时会进入到`set_user_nice()`。

```c
void set_user_nice(struct task_struct *p, long nice)
{
    /* rt or deadline task */
    if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
        p->static_prio = NICE_TO_PRIO(nice);
        goto out_unlock;
    }
    /* cfs task */
    p->static_prio = NICE_TO_PRIO(nice);
    set_load_weight(p, true);
    p->prio = effective_prio(p);
}
```

虽然`static_prio`仅用于Fair-Task，但是设置一个Deadline-Task或者Realtime-Task的`static_prio`是被允许的，并且这并不会产生实质影响，因为`rt_sched_class`和`dl_sched_class`的调度决策并不参考这个优先级。对于Fair-Task来说`set_user_nice()`除了更新`static_prio`以外还会调用`set_load_weight()`更新Fair-Task以及所在的`cfs_rq`的权重信息，CFS调度器进行调度决策时依赖的是权重值而不是Nice值（Nice值和权重之间存在转化关系）。

`set_load_weight()`的代码如下，在CFS中有一类调度策略为`SCHED_IDLE`的特别任务，这些任务参与CFS的任务调度但是仅占用非常少的CPU时间（通过将这些任务的权重设置为一个固定的最小值实现），因此对于使用`SCHED_IDLE`调度策略的Task修改其Nice值并不会影响调度决策。

```c
static void set_load_weight(struct task_struct *p, bool update_load)
{
    int prio = p->static_prio - MAX_RT_PRIO;
    struct load_weight *load = &p->se.load;
    /* idle policy */
    if (idle_policy(p->policy)) {
        load->weight = scale_load(WEIGHT_IDLEPRIO);
        load->inv_weight = WMULT_IDLEPRIO;
        return;
    }
    /* normal cfs task */
    reweight_task(p, prio);
}
```

## rt_priority

`rt_priority`是实时调度器使用的优先级，系统调用`sched_setscheduler()`支持修改任务的调度类的同时可以设置`rt_priority`，如果是切换到CFS调度类该参数是无效的，输入的`rt_priority`的有效范围为`[0,99]`，与Nice值不同的是`rt_priority`越大其Realtime-Task的优先级就越大，但是内核为了让语义统一在使用时做了如下转化: `real-prio = MAX_RT_PRIO - 1 - rt_priority = 99 - rt_priority`，`rt_priority`越大实际使用的优先级值越小。

## normal_prio

`normal_prio`的值与当前所属的调度策略有关，Deadline-Task的`normal_prio`固定`-1`，Realtime-Task的`normal_prio`为`99 - rt_priority`，Fair-Task的`normal_prio`为`static_prio`。`normal_prio`实现了`static_prio`值和`rt_priority`优先级的统一，`normal_prio`值越小其优先级越高。

## prio

`prio`是做调度决策时真正使用的优先级，在正常情况下与`normal_prio`一致。但是在一些特殊场景下需要在不修改`static_prio`和`rt_priority`的情况下临时提高`prio`。

典型的就是优先级反转问题，比如在以下两个场景就需要临时提高`prio`：

* Realtime-Task持有`mutex A`，另一个更高优先级的Deadline-Task阻塞在了`mutex A`上。
* Realtime-Task持有`mutex A`，另一个同优先级但是满足抢占条件的Realtime-Task阻塞在了`mutex A`上。

在这里持锁的Task称作`p`，被阻塞的Task称作`pi`。如果不做处理，某个中间优先级的Task有可能会抢占`p`执行，导致因为锁阻塞的`pi`任务的实时性得不到保障。解决方法就是通过优先级继承在持锁过程中让`p`的`prio`和`pi`的`prio`保持一致，这样`p`就能够尽快执行结束后释放`mutex`避免`pi`被阻塞时间过长。

## 总结

四类优先级的范围与用途一览：

| 类型          | 范围        | 用途                   | 设置方法                                                                                                                                                  |
| ------------- | ----------- | ---------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `static_prio` | `[100,139]` | Fair-Task优先级    | 通过`set_priority`系统调用设置，将输入的Nice值从`[-20,19]`映射到`[100,139]`，Nice值越高优先级越低                                  |
| `rt_priority` | `[0,99]`    | Realtime-Task优先级        | 通过`sched_setscheduler`系统调用设置， 输入范围为`[0,99]`，值越高优先级越高                                                                               |
| `normal_prio` | `[-1,139]`  | 优先级的语义统一       | `normal_prio`与调度类相关，`fair_sched_class`与`static_prio`一致，`rt_sched_class`将`rt_priority`从`[0,99]`映射到`[99,0]`，`dl_sched_class`固定为`-1` |
| `prio`        | `[-1,139]`  | 决策时真正使用的优先级 | 一般与`normal_prio`一致，特殊情况可以临时提高优先级                                                                                                       |

从优先级机制的设计上来看，实际调度时会按先后顺序考虑`prio`、调度类之间的优先级、调度类内部的优先级。临时提高的特殊情况下可以不考虑任务当前的调度类与优先级参数设置。普通情况下使用`normal_prio`，而`normal_prio`会根据当前的调度类和优先级参数设置进行调整。

设置优先级相关的系统调用：

* `set_priority`可以改变`static_prio`的值，输入参数为Nice
* `sched_setscheduler`可以设置调度器以及`rt_priority`（如果调度类为`rt_sched_class`）。输入参数为`rt_priority`的值以及调度策略`policy`（可找到对应的调度器）。

设置优先级相关的Linux命令:

* `renice`可以调整Fair-Task的Nice值。
* `chrt`可以设置调度策略，如果是Realtime-Task可以设置`rt_priority`
* `cat /proc/{pid}/sched` 可以查看`prio`和`policy`
