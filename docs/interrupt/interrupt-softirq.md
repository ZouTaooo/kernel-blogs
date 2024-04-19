# softirq

软中断（`softirq`）是内核虚拟出的一种异步中断，通过`raise_softirq()`来触发，可以将一些不紧急的任务推迟执行。在软中断中可以处理中断下半部，比如网卡数据收发的软中断`NET_TX_SOFTIRQ`和`NET_RX_SOFTIRQ`，还可以处理一些需要异步执行的场景，比如定时器软中断`TIMER_SOFTIRQ`。`softirq`这种异步处理机制能够避免阻塞高优先级的任务执行（如中断处理程序），保障系统的实时性。

## 软中断号

```c
// include/linux/interrupt.h
enum
{
    HI_SOFTIRQ=0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    IRQ_POLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ,
    RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

    NR_SOFTIRQS
};
```

软中断具有两个特点：

* 软中断上下文是没有调度实体的，因此`softirq`在执行过程中无法睡眠。
* 软中断可以并行（同一个软中断处理函数可以在多核上同时执行），这意味着`softirq`具有高性能的优势，但是编写一个支持并行的软中断处理函数需要非常小心，因此目前只有网络、定时器等对性能要求高的场景使用软中断。
  
由于软中断的特点内核中只静态定义了10个软中断，并且不建议增加新的软中断，对于绝大多数场景可以使用不存在并发的`tasklet`作为替代品。

## 触发软中断
```c
// kernel/softirq.c
DEFINE_PER_CPU_ALIGNED(irq_cpustat_t, irq_stat);
EXPORT_PER_CPU_SYMBOL(irq_stat);

typedef struct {
    u16	     __softirq_pending;
    ......
} ____cacheline_aligned irq_cpustat_t;

```

内核定义了一个`per-cpu`的结构体`irq_cpustat_t`，这个结构体中存放了很多中断相关的统计信息，以及一个软中断的`pending`位图`__softirq_pending`，位图的每一位对应一个软中断号，如果某`bit`为`1`表示该软中断需要处理。

```c
// kernel/softirq.c
void raise_softirq(unsigned int nr)
{
    unsigned long flags;

    local_irq_save(flags);
    raise_softirq_irqoff(nr);
    local_irq_restore(flags);
}

// 必须在关中断的上下文调用
inline void raise_softirq_irqoff(unsigned int nr)
{
    __raise_softirq_irqoff(nr);
    
    if (!in_interrupt())
        wakeup_softirqd();
}

// 设置位图
void __raise_softirq_irqoff(unsigned int nr)
{
    or_softirq_pending(1UL << nr);
}

```

`raise_softirq()`和`raise_softirq_irqoff()`都可以触发一个编号为`nr`的软中断，只是`raise_softirq_irqoff()`在关中断的上下文中调用。

在`__raise_softirq_irqoff()`中会设置软中断号对应的`pending`位，然后会检查当前是否处于中断上下文（硬中断上下文+软中断上下文+中断下半部禁用上下文），如果此时不处于中断上下文，为了软中断能够尽快执行会唤醒`ksoftirqd`线程参与调度，如果此时已经在中断上下文了，之后从中断上下文返回到进程上下文时会尝试执行软中断。

## 软中断处理

软中断有两种执行方式：

* 从中断上下文退出时会尝试处理软中断。这又分为从硬中断上下文和中断下半部禁用状态退出两种情况，此时需要检查是否仍然处于中断状态（可能存在中断嵌套、下半部禁用嵌套或者中断抢占软中断），如果此时即将返回进程上下文则尝试处理软中断。
* 如果在中断下半部执行时软中断数量过多或者在进程上下文触发一个软中断，此时会唤醒`ksoftirqd`线程参与调度，执行软中断。
> 从软中断上下文回到进程上下文时由于刚处理完软中断此时不会再次尝试执行软中断。

## 从中断下半部禁用回到进程上下文


```c
// include/linux/bottom_half.h
static inline void local_bh_enable(void)
{
    __local_bh_enable_ip(_THIS_IP_, SOFTIRQ_DISABLE_OFFSET);
}

// kernel/softirq.c
void __local_bh_enable_ip(unsigned long ip, unsigned int cnt)
{
    ......
    preempt_count_sub(cnt - 1);
    if (unlikely(!in_interrupt() && local_softirq_pending())) {
        do_softirq();
    }
    preempt_count_dec();
    preempt_check_resched();
}
EXPORT_SYMBOL(__local_bh_enable_ip);
```

当退出中断下半部禁用临界区时会调用`local_bh_enable()`，参考`preempt_count`的位划分，`SOFTIRQ_DISABLE_OFFSET`是`0x200`,
在`__local_bh_enable_ip()`函数中先将`preempt_count`减去了`SOFTIRQ_DISABLE_OFFSET - 1`，这等价于先跳出一层中断下半部禁用，然后禁用了抢占，因为之后还要检查是否有软中断是否需要处理，当没有软中断需要处理或者等到软中断处理结束以后再将抢占计数减一启用抢占。这是因为软中断执行过程中是不允许抢占的，有以下两个原因：
* 不禁用抢占会出现异常：`do_softirq()`会先获取本地CPU的`pending`位图之后再设置软中断标记（可以达到禁用抢占和禁用软中断嵌套的效果），如果在这个过程中发生了抢占，等到下次任务被调度时可能已经不在之前执行的`CPU`上了，但是此时处理的`pending`位图是上一个`CPU`的。
* 软中断优先级高于进程：`do_softirq()`有权利在软中断过多时主动放弃`CPU`，将剩余软中断交给`ksoftirqd`线程去处理，但是进程没有资格去抢占软中断执行。

## 从硬中断上下文回到进程上下文

```c
// kernel/softirq.c
void irq_exit(void)
{
    __irq_exit_rcu();
    ......
}

static inline void __irq_exit_rcu(void)
{
    ......
    preempt_count_sub(HARDIRQ_OFFSET);
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();
    ......
}
```

离开硬中断上下文时需要调用`irq_exit()`，在`__irq_exit_rcu()`中会将硬中断计数减一，然后判断当前是否不处于中断上下文中（有可能当前硬中断抢占发生在软中断执行过程或者是中断下半部禁用上下文）并且有待处理的软中断，此时调用`invoke_softirq()`处理。


```c
// kernel/softirq.c
static inline void invoke_softirq(void)
{
    if (ksoftirqd_running(local_softirq_pending()))
        return;

    if (!force_irqthreads) {
#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
        __do_softirq();
#else
        do_softirq_own_stack();
#endif
    } else {
        wakeup_softirqd();
    }
}
```

`invoke_softirq()`会检查`ksoftirqd`线程是否已经在运行了，如果已经处于运行状态就交给`ksoftirqd`去处理软中断。`force_irqthreads`判断是否强制进行中断线程化，如果有这样的要求则唤醒`ksoftirqd`线程去处理软中断，否则此时就开始处理软中断。

如果当前`Arch`下有单独的中断栈，由于此时即将退出最后一层硬中断上下文，栈的空间很充足，可以直接在中断栈上执行软中断。如果当前`Arch`下没有单独的中断栈，则会在当前进程的内核栈上执行。

## 软中断执行

从中断下半部禁用和硬中断上下文返回进程上下文时，都会调用`__do_softirq()`处理软中断。


```c
// kernel/softirq.c
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
    int max_restart = MAX_SOFTIRQ_RESTART;
    struct softirq_action *h;
    __u32 pending;
    int softirq_bit;

    pending = local_softirq_pending();
    __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
    
restart:
    // 处理阶段
    set_softirq_pending(0);
    local_irq_enable();
    h = softirq_vec;
    while ((softirq_bit = ffs(pending))) {
        h += softirq_bit - 1;
        h->action(h);
        h++;
        pending >>= softirq_bit;
    }
    local_irq_disable();

    // 检查是否重启
    pending = local_softirq_pending();
    if (pending) {
        if (time_before(jiffies, end) && !need_resched() &&
            --max_restart)
            goto restart;

        wakeup_softirqd();
    }
    __local_bh_enable(SOFTIRQ_OFFSET);
}
```

在处理软中断前会标记当前处于软中断上下文，然后开启中断，允许被中断抢占。处理阶段会遍历每一个在`pending`中的软中断，执行对应的软中断处理函数。每一轮处理结束时会关闭中断，检查在软中断处理过程中是否触发了新的软中断（硬中断处理函数内部可能会触发软中断），如果有则检查是否满足继续处理下去的条件，如果满足以下三个条件中的任意一个则停止处理，唤醒`ksoftirqd`线程参与调度。

* 总的处理时间超过了`2ms`（`MAX_SOFTIRQ_TIME`）
* 设置了抢占标记`TIF_NEED_RESCHED`
* 超过了处理`10`轮