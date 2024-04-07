<!-- task_struct中prio&static_prio&normal_prio&rt_prioity -->
## 前言

在阅读源码的过程中发现一个`task_struct`包含四个优先级相关的成员，`prio`、`static_prio`、`normal_prio`和`rt_priority`这几个优先级值有什么区别和联系呢？
```c
struct task_struct {
    int				prio;
    int				static_prio;
    int				normal_prio;
    unsigned int    rt_priority;
}
```

## 内核中的优先级范围

```c
+----+-------------------------+---------------+
| -1 |0                      99|100         139|   
+----+-------------------------+---------------+
```

内核中管理多任务的调度类有三个，分别是`dl_sched_class`、`rt_sched_class`、`fair_sched_class`。其中`-1`优先级用于所有的`dl-task`。`[0,99]`是`rt-task`的优先级，`[100,139]`是`normal-task`的优先级。在内核中值越低其优先级越高。

## static_prio

在内核中，`static_prio`用于`normal-task`，这个值与`nice`存在如下转化关系：`static_prio = nice + 120`。可以看到`static_prio`与`nice值`是绑定的。
内核中设置一个进程的优先级的入口是系统调用`sys_setpriority`，这个系统调用可以设置用户、进程组、进程的优先级。当设置某个进程的`nice`值时会进入到`set_user_nice`。
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
如果设置一个deadline任务或者`rt-task`的`static_prio`时，该操作是允许的，但是并不会产生实质影响，因为`rt_sched_class`和`dl_sched_class`并不看这个值。对于`normal-task`来说`set_user_nice`除了更新`static_prio`以外，`set_load_weight`还会更新`normal-task`以及`cfs_rq`的权重信息。

CFS中有一类特别的任务，这些任务的调度策略为`SCHED_IDLE`，这些任务的权重值与优先级无关，因此对于使用该策略的`task`修改其`nice`值没有意义。
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
以上只是`static_prio`产生变化的一种场景，一个任务的生命周期内`static_prio`的变化的情况有两种：
- 系统调用`fork`：在新任务初始化时，`sched_fork`会重置或者继承。
- 系统调用`sys_setpriority`中，调用`set_user_nice`更新。

**NOTE**：系统调用`sched_setscheduler`支持修改任务的调度类的同时可以设置优先级，但是该参数是实时调度器使用的优先级，对于切换到CFS的场景，此时更新权重信息使用的是任务自身的`static_prio`。这样来看`sys_setpriority`是修改`static_prio`的唯一接口。

## rt_priority

`rt_priority`是实时调度器使用的优先级，在设置一个`rt-task`的`rt_priority`时，输入值的有效范围为`[0,99]`，与`nice`值不同的是`rt_priority`越大其`rt-task`的优先级就越大，但是内核为了让语义统一在使用时做了转化。`real-prio = MAX_RT_PRIO - 1 - rt_priority = 99 - rt_priority`，优先级越大其计算出的值越小。但是在外部使用时可以按照正常逻辑来设置优先级。

## normal_prio

`normal_prio`的值与当前所属的调度策略有关，`dl-task`的`normal_prio`固定`-1`，`rt-task`的`normal_prio`为`99 - rt_priority`，`normal-task`的`normal_prio`为`static_prio`。`normal_prio`实现了`static_prio`值和`rt_priority`优先级的统一，`normal_prio`值越小其优先级越高。

## prio

`prio`是做调度决策时真正使用的优先级，在正常情况下与`normal_prio`一致。但是在一些特殊场景下需要在不修改`static_prio`和`rt_priority`的情况下临时提高`prio`。

典型的就是优先级反转问题，在以下两个场景需要临时提高`prio`：
- `rt-task`持有`mutex A`，另一个`dl-task`阻塞在`mutex A`上。
- `rt-task`持有`mutex A`，另一个`rt-task`阻塞在`mutex A`上，并且该任务可以抢占当前运行中的进程。

在这里持锁的`task`称作`p`，阻塞的`task`称作`pi`。如果不做处理，某个中间优先级的`task`有可能会抢占`p`执行，导致因为锁阻塞的`pi`任务的实时性得不到保障。解决方法就是通过优先级继承让`p`持锁过程中`prio`和`pi`的`prio`保持一致，让`p`能够尽快执行结束后释放`mutex`。

## 总结
四类优先级的范围与用途一览：
| 类型          | 范围        | 用途                   | 设置方法                                                                                                                                                  |
| ------------- | ----------- | ---------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `static_prio` | `[100,139]` | `normal-task`优先级    | 通过`set_priority`系统调用设置nice值转化为`static_prio`，将`nice`值从`[-20,19]`映射到`[100,139]`，`nice`值越高优先级越低                                  |
| `rt_priority` | `[0,99]`    | `rt-task`优先级        | 通过`sched_setscheduler`系统调用设置， 输入范围为`[0,99]`，值越高优先级越高                                                                               |
| `normal_prio` | `[-1,139]`  | 优先级的语义统一       | `normal_prio`与调度类对应，`fair_sched_class`时与`static_prio`一致，`rt_sched_class`时将`rt_priority`从`[0,99]`映射到`[99,0]`，`dl_sched_class`时固定`-1` |
| `prio`        | `[-1,139]`  | 决策时真正使用的优先级 | 一般与`normal_prio`一致，特殊情况可以临时提高优先级                                                                                                       |

从优先级机制的设计上来看，真正的优先级`prio`会按顺序考虑优先级临时提高、调度类、调度类对应的优先级设置。临时提高的特殊情况下可以不考虑任务当前的调度类与优先级参数设置。普通情况下使用`normal_prio`，而`normal_prio`会根据当前的调度类和参数设置进行调整。

另外在API上，设置优先级的接口有两个：
- `set_priority`可以改变`static_prio`的值，输入参数为`nice`
- `sched_setscheduler`可以设置调度器以及`rt_priority`（如果调度类为`rt_sched_class`）。输入参数为`rt_priority`的值以及调度策略`policy`（可找到对应的调度器）。

相关的linux命令有:
- `renice`可以调整`normal-task`的`nice`值。
- `chrt`可以设置调度策略，如果是`rt-task`可以设置`rt_priority`
- `cat /proc/{pid}/sched` 可以查看`prio`和`policy`