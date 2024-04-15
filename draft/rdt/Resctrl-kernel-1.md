<!-- #! https://zhuanlan.zhihu.com/p/647039275 -->
<!-- # Resctrl内核实现（一）CPU状态 -->
## 前言

resctrl是rdt机制的一个用户态接口，通过对rdt技术进行封装，提供了一套资源分配和监控机制的接口，方便用户进行使用。本文从resctrl的资源分配和监控的角度对内核源码实现进行了分析，参考的kernel版本为4.19.287。

## Resctrl下的CPU状态变化

resctrl中有两种group，控制组和监控组，控制组（CTRL group）制定资源分配策略，监控组（mon group）负责进行指标采集，而resctrl中将控制组和监控组进行了组合，被称作CTRL-MON group。resctrl根目录就是一个CTRL-MON group，这种group兼顾了资源分配规则和监控。
对RDT技术有一定了解的同学会知道RDT技术的关键就在RMID和CLOSID，那么在resctrl中RMID和CLOSID时怎么管理的呢？cpu在不同的时刻会运行不同task，而resctrl为了支持复杂的资源分配和监控逻辑，cpu在不同的时刻会使用不同的RMID和CLOSID。下面这个结构体是per-cpu的，每个cpu都拥有`default_rmid`、`default_closid`以及`cur_rmid`、`cur_closid`四个ID。这个设计非常关键，正是这个结构体实现了resctrl的核心逻辑。

```c
struct intel_pqr_state {
    u32   cur_rmid;
    u32   cur_closid;
    u32   default_rmid;
    u32   default_closid;
};
```

我们知道RMID用于决定当前的cpu上发生事件时应该累积到哪个监控组去，而CLOSID让当前cpu去查询对应的CBM（cache bit mask）和DV（delay value），限制thread去访问规定范围内的资源同时限制内存带宽总量。`cur_rmid`和`cur_closid`就存储了当前时刻cpu的IA32_PQR_ASSOC寄存器里的closid和rmid值。`cur_rmid`和`cur_closid`是会随着task的切换改变的，当cpu换入的task被指定了RMID或者CLOSID时，就会更新对应的`cur_rmid`或`cur_closid`为task指定的ID，需要注意这里用到了或这个字，这是因为**可以给task单独指定rmid而不指定closid或者只指定closid而不指定rmid**；当cpu换入的task没有指定rmid和closid时cpu就会使用per-cpu的`default_rmid`和`default_closid`。
在知道了`cur_rmid`和`cur_closid`的切换规则之后，这里涉及到了两个问题，在resctrl中：

1. per-cpu的`default_rmid`和`default_closid`是如何指定的？
2. per-task的RMID和CLOSID是如何指定的？什么时候task的RMID和CLOSID是未指定的？

对于第一个问题，了解resctrl文件系统的同学知道每一个CTRL-MON group目录下都有一个cpus_list文件存放着cpu的掩码，很容易想到cpu的`default_closid`就是所属的CTRL-MON group。而`default_rmid`就要复杂一些，因为在resctrl有默认监控组（CTRL-MON group中配套的监控组）也有mon_groups目录下的监控组，如果将task pid写入到非默认的MON group的tasks文件中，此时会指定使用MON group的RMID否则使用所属CTRL-MON group的RMID。

resctrl中的每个CTRL-MON group和MON group都被分配了一个CLOSID和RMID，每个cpu都只能在一个CTRL-MON group的`cpu_lists`中有效，因此cpu的`default_rmid`和`default_closid`就是所属的CTRL-MON group的rmid和closid，由于CTRL-MON group下可以新建MON-group，当cpu被分配到当前CTRL-MON group中的MON-group中去时，cpu的`default_rmid`会更新为对应MON group的rmid。

在充分理解了以上closid和rmid的分配与使用规则后，就能够很好理解resctrl中task的资源使用和监控规则。
**资源分配规则**：

1. 当一个task加入一个非默认的CTRL-MON group时，此时task在运行中时，不管在哪个cpu上都受到非默认CTRL-MON group的schemata限制；
2. 如果一个task属于默认CTRL-MON group，也就是没有为task指定closid，此时如果运行的cpu属于一个非默认CTRL-MON group，按照非默认CTRL-MON group的schemata的限制访问资源。其实说白了也就是在哪个cpu上按照哪个cpu的默认规矩来；
3. 其他情况，按照默认CTRL-MON group的schemata访问资源。
**监控规则**：
1. 当一个task加入了一个mon_groups目录下的MON group或者属于一个非默认的CTRL-MON group时，此时关于task的rdt事件会报告到所属的MON-group中；
2. 如果一个task不属于某个MON group，同时不属于某个非默认的CTRL-MON group，此时按照运行的cpu进行划分，如果当前cpu属于非默认CTRL-MON group，则报告到非默认CTRL-MON group；
3. 其他情况，报告给默认CTRL-MON group的监控组。
**Note**：默认CTRL-MON group指的就是根目录，默认MON- group就是每个CTRL-MON group配套的监控组。

看完这段规则说明的时候正常人是很难理解的，觉得非常烧脑。但其实这段规则从底层逻辑上来解释非常简单：per-cpu的`cur_rmid`和`cur_closid`决定了RDT机制此时发挥作用的资源分配策略以及事件累积监控组。

所以搞清楚task切换时`cur_rmid`和`cur_closid`的切换规则就迎刃而解。

```c
// 如果当前task有closid和rmid，按照task的closid和rmid更新pqr assoc msr寄存器
// 并且更新per-cpu的intel_pqr_state结构体
static void __intel_rdt_sched_in(void)
{
    // intel_pqr_state是pqr msr寄存器的值，包括当前cpu上的
    struct intel_pqr_state *state = this_cpu_ptr(&pqr_state);
    u32 closid = state->default_closid;
    u32 rmid = state->default_rmid;
    
    /*
    * If this task has a closid/rmid assigned, use it.
    * Else use the closid/rmid assigned to this cpu.
    */
    if (static_branch_likely(&rdt_alloc_enable_key)) {
        if (current->closid)
            closid = current->closid;
    }

    if (static_branch_likely(&rdt_mon_enable_key)) {
        if (current->rmid)
            rmid = current->rmid;
    }

    if (closid != state->cur_closid || rmid != state->cur_rmid) {
        state->cur_closid = closid;
        state->cur_rmid = rmid;
        wrmsr(IA32_PQR_ASSOC, rmid, closid);
    }
}
```

在内核中当task换入时会调用`__intel_rdt_sched_in`函数，可以看到`cur_rmid`和`cur_closid`和变换规则如下：

1. 如果task指定了rmid和closid则使用task的rmid和closid
2. 如果task未指定则使用cpu的`default_rmid`和`default_closid`

逻辑就这么简单，当我们清楚当前配置下每个cpu的`default_rmid`和`default_closid`、每个task是否指定了RMID和CLOSID后就能清楚理解资源配置策略的覆盖对象以及事件监控的包含对象。
