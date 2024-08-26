<!-- 时间子系统 -->

## 前言

调度算法决策每个任务什么时候开始执行执行多久，因此调度器的决策依赖于时间子系统对时间的衡量，同时调度逻辑的触发和时钟的联系也很紧密，所以需要认真的学习一下时钟子系统的基本结构以及相关的概念。

## 基础概念
### HZ&tick

HZ表示1s内出现时钟中断的次数，HZ是一个在内核中预定义的值，不同的CPU上使用的HZ可能是不同的，常见的有100、500和1000。tick的周期由表示1/HZ表示，比如HZ=100，则每次tick的间隔为(1/100)s=10ms，HZ=1000，则周期为(1/1000)s=1ms。高HZ会带来以下几个好处：

* 内核定时器的精度更高。
* select和poll等基于超时的系统调用性能更好。
* 系统状态指标的测量，比如uptime等命令更精准。
* 调度延迟更小，由于等待tick导致的进程过度运行的平均时长和出现次数更少，抢占发生的更及时。

这都是由于tick发生的更频繁，从而处理抢占、定时器、指标统计等操作也发生的更频繁，也因此提高HZ会让更多的资源消耗在处理这些周期性事务上，HZ从100提高到1000，资源开销提高了大约10倍。但是一般来说，100，500，1000等HZ值的资源开销都是可接受的。

**NOTE**: 可以通过`getconf CLK_TCK`命令查看系统当前的HZ值。

### jiffies

`jiffies`记录了从boot启动到

### clock_source

什么是clock_source，clock_source的作用，clock_source结构体，clock_jiffies和refined_jiffies。

### watch_dog

## 时钟管理框架

## Reference

* [The Tick Rate: HZ](https://litux.nl/mirror/kerneldevelopment/0672327201/ch10lev1sec2.html)
* [timekeeping](https://www.kernel.org/doc/Documentation/timers/timekeeping.txt)
* [timers and time management](https://0xax.gitbooks.io/linux-insides/content/Timers/)
* [蜗窝科技-时间子系统系列](http://www.wowotech.net/sort/timer_subsystem)