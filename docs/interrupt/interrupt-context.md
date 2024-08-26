# 中断上下文

## 前言

中断处理过程中会禁止抢占，如果中断处理函数不能尽快执行完成就会影响系统的实时性，为此内核将中断的处理过程分为上半部(`Top Half`)和下半部(`Bottom Half`)，将耗时操作推迟到下半部异步执行。

* 中断上半部：在上半部中执行一些能够快速完成的动作，比如响应外设请求。
* 中断下半部：在下半部中执行一些耗时的动作，比如网络协议栈解析

因此在内核中CPU就有可能处于三种上下文：

* 硬中断上下文：CPU在执行中断上半部时处于硬中断上下文，在较新的内核中断设计中，硬中断上下文会禁用中断，因此不会产生中断嵌套，只有等待当前中断上半部执行结束后才能响应下一个中断。
* 软中断上下文：CPU在执行中断下半部时处于软中断上下文，在软中断上下文中可以被中断抢占，但是在CPU上软中断不允许嵌套，如果在执行中断下半部时发生中断，在处理完新的中断上半部之后不会进入新的软中断上下文。
* 进程上下文：CPU在执行进程代码时处于进程上下文

内核使用了一个`per-pcu`的变量`preempt_count`来区分这些上下文，同时管理中断嵌套、抢占行为。

## preempt_count

`preempt_count`是一个`32 bits`的整数，`32 bits`被划分为如下五个部分:

* `bits[0,8)`: `preempt disable count`抢占禁用计数占`8 bits`，执行`preempt_disable()`会对抢占禁用计数加一，`preempt_disable()`和`preempt_enable()`必须要成对使用。当抢占禁用计数不为`0`时禁用抢占，此时不会发生调度，内核可以支持`255`层的抢占禁用嵌套。
* `bit 8`: `softirq count`软中断计数，当`softirq count`不为`0`时表示软中断正在执行，由于不会出现软中断嵌套因此只需要一个`bit`表示。
* `bits[9,16)`: `bottom half disable count`中断下半部禁用计数，进程有时会有同步需要，此时不允许被软中断抢占，`local_bh_disable()`将计数加一，`local_bh_disable()`和`local_bh_enable()`需要成对使用。可以支持`127`层中断下半部禁用嵌套。
* `bits[16,20)`: `hardirq count`硬中断计数占用`4 bits`，当硬中断计数不为`0`时表示硬中断正在执行，之前提到过内核不会出现硬中断嵌套，理论上只用一个`bit`就足够了，但是为了兼容一些有古董驱动（极少数情况），这些驱动可能在中断处理过程中打开中断，发生中断嵌套。内核能够支持最多`15`层硬中断嵌套，用于处理这些过时驱动。
* `bits[20,24)`: `NMI count`不可屏蔽中断计数占用`4 bits`。处理`NMI`中断时需要将硬中断计数和`NMI`中断计数都加一。
* `bits[24,31)`: 未使用
* `bit 31`: `PREEMPT_NEED_RESCHED`用于表明内核有更高优先级的任务需要被调度，当抢占禁用计数不为`0`时才会设置，但是我看这个`bit`在调度中并没有发挥实际作用，内核中有对这个`bit`设置和清楚，但是在调度决策过程中并没有读取这个`bit`值。

```c
/*
 *         PREEMPT_MASK:	0x000000ff
 *         SOFTIRQ_MASK:	0x0000ff00
 *         HARDIRQ_MASK:	0x000f0000
 *             NMI_MASK:	0x00f00000
 * PREEMPT_NEED_RESCHED:	0x80000000
 */

// bits number
#define PREEMPT_BITS	8
#define SOFTIRQ_BITS	8
#define HARDIRQ_BITS	4
#define NMI_BITS	4

// field shift
#define PREEMPT_SHIFT	0
#define SOFTIRQ_SHIFT	(PREEMPT_SHIFT + PREEMPT_BITS)
#define HARDIRQ_SHIFT	(SOFTIRQ_SHIFT + SOFTIRQ_BITS)
#define NMI_SHIFT	(HARDIRQ_SHIFT + HARDIRQ_BITS)

// field mask
#define __IRQ_MASK(x)	((1UL << (x))-1)
#define PREEMPT_MASK	(__IRQ_MASK(PREEMPT_BITS) << PREEMPT_SHIFT)
#define SOFTIRQ_MASK	(__IRQ_MASK(SOFTIRQ_BITS) << SOFTIRQ_SHIFT)
#define HARDIRQ_MASK	(__IRQ_MASK(HARDIRQ_BITS) << HARDIRQ_SHIFT)
#define NMI_MASK	(__IRQ_MASK(NMI_BITS)     << NMI_SHIFT)

// field offset
#define PREEMPT_OFFSET	(1UL << PREEMPT_SHIFT)
#define SOFTIRQ_OFFSET	(1UL << SOFTIRQ_SHIFT)
#define HARDIRQ_OFFSET	(1UL << HARDIRQ_SHIFT)
#define NMI_OFFSET	(1UL << NMI_SHIFT)

// irq count
#define hardirq_count()	(preempt_count() & HARDIRQ_MASK)
#define softirq_count()	(preempt_count() & SOFTIRQ_MASK)
#define irq_count()	(preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK \
                 | NMI_MASK))

/* 
 * in_irq()       - We're in (hard) IRQ context
 * in_softirq()   - We have BH disabled, or are processing softirqs
 * in_interrupt() - We're in NMI,IRQ,SoftIRQ context or have BH disabled
 * in_serving_softirq() - We're in softirq context
 * in_nmi()       - We're in NMI context
 * in_task()	  - We're in task context
 */
#define in_irq()		(hardirq_count())
#define in_softirq()		(softirq_count())
#define in_interrupt()		(irq_count())
#define in_serving_softirq()	(softirq_count() & SOFTIRQ_OFFSET)
#define in_nmi()		(preempt_count() & NMI_MASK)
#define in_task()		(!(preempt_count() & \
                   (NMI_MASK | HARDIRQ_MASK | SOFTIRQ_OFFSET)))
```

通过`preempt_count`可以判断当前所处的上下文。需要注意软中断上下文有两个检查函数：

* `in_softirq()`取`bits[8,16)`进行判断是否是软中断上下文，因此宏观上中断下半部禁用也可以被认为是软中断上下文。
* `in_serving_softirq()`判断当前是否处于中断下半部执行过程中，可以取`softirq count`这个`bit`位进行确认。
