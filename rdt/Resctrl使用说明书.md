<!-- #! https://zhuanlan.zhihu.com/p/647038545 -->
<!-- # Resctrl使用说明书 -->
## 前言

Resctrl文件系统是Linux内核在4.10提供的对RDT技术的支持，作为一个伪文件系统在使用方式上与cgroup是类似，通过提供一系列的文件为用户态提供查询和修改接口。本文就resctrl文件系统的使用进行了详细说明，内容基本来自于Linux Documentation中的精华部分。

## 使用限制与挂载

### 检查

resctrl的使用存在两个限制：

1. 内核版本4.10+
2. cpu提供rdt能力，检查是否支持可以查看/proc/cpuinfo文件，查看flags是否包含以下特性

|Feature| flag|
| --- | --- |
|RDT (Resource Director Technology) Allocation| "rdt_a"|
|CAT (Cache Allocation Technology)| "cat_l3", "cat_l2"|
|CDP (Code and Data Prioritization)| "cdp_l3", "cdp_l2"|
|CQM (Cache QoS Monitoring)| "cqm_llc", "cqm_occup_llc"|
|MBM (Memory Bandwidth Monitoring)| "cqm_mbm_total", "cqm_mbm_local"|
|MBA (Memory Bandwidth Allocation) |"mba"|
|SMBA (Slow Memory Bandwidth Allocation) |""|
|BMEC (Bandwidth Monitoring Event Configuration)| ""|

### 挂载

```bash
mount -t resctrl resctrl /sys/fs/resctrl
```

## 目录与文件

### info目录说明

`/sys/fs/resctrl`是resctrl的根目录，根目录本身就是一个rdt_group，关于rdt_group在下文进行描述。除了rdt_group本身包含的文件和目录外根目录下有一个info目录包含了resctrl和rdt的一些信息。

```bash
info
├── L3 
│   ├── bit_usage 
│   ├── cbm_mask # cache bit mask 
│   ├── min_cbm_bits # cbm的最小连续长度
│   ├── num_closids # closid的个数
│   └── shareable_bits
├── L3_MON
│   ├── max_threshold_occupancy # 当某rmid的llc占用量计数器低于该值时考虑释放，与rmid重用有关
│   ├── mon_features # 支持的监控事件列表
│   └── num_rmids # rmid的数量
├── last_cmd_status # 最后一条指令执行的结果
└── MB
    ├── bandwidth_gran # 带宽的设置的粒度
    ├── delay_linear 
    ├── min_bandwidth # 最小的内存带宽百分比
    ├── num_closids # closid数量
    └── thread_throttle_mode # core上的多个thread存在不同带宽时的处理，max-以最大限制同时压制 per-thread：各自应用不同的带宽比例
```

### rdt_group目录

一个rdt_group包含的文件如下：

```bash
├── cpus # cpu bit mask
├── cpus_list # 和cpus意义相同，只是使用范围表示
├── id
├── mode
├── mon_data # 监控的数据目录
├── mon_groups # 监控组目录
├── schemata # 资源配置策略
├── size # 和schemata一样，只是schemata使用bit mask，size使用数字
└── tasks # pids
```

### mon_data目录

在rdt_group下的mon_data目录存放的是属于该rdt_group的默认监控组的事件value。

```bash
├── mon_L3_00
│   ├── llc_occupancy
│   ├── mbm_local_bytes
│   └── mbm_total_bytes
└── mon_L3_01
    ├── llc_occupancy
    ├── mbm_local_bytes
    └── mbm_total_bytes
```

`00`和`01`标记的是多个socket。

### mon_groups目录

每一个rdt_group下都有一个mon_groups目录，除了默认的监控组数据会存放在mon_data目录下以外，还可以在mon_groups目录下新建mon_group实现更细粒度的监控。

```bash
/sys/fs/resctrl/mon_groups/
└── mon_demo
    ├── cpus
    ├── cpus_list
    ├── id
    ├── mon_data
    └── tasks
```

和rdt_group一样的是可以指定cpus，以及tasks。

## 常见用法

### 创建rdt_group

rdt_group只能够创建在根目录下，不允许嵌套，这是和cgroup的一个区别。只需要通过mkdir创建新目录即可。

```bash
mkdir /sys/fs/resctrl/rdt_group_demo
```

### rdt_group设置资源分配规则

修改schemata文件就可以改变rdt_group的资源访问策略。

```bash
# cat  /sys/fs/resctrl/demo_rdt_group/schemata 
MB:0=100;1=100
L3:0=fff;1=fff
# echo MB:0:70;1=80
# echo L3:0=ff0;1=0ff
```

需要注意的是MB指的是内存带宽的百分比，会收到info目录下的最小带宽比以及最小带宽粒度的限制。所有不满足最小带宽粒度的值都会被取整到满足带宽粒度。
L3表示llc的掩码，fff表示cache被划分为12份，每一个bit表示1/12。需要注意的是L3的掩码必须连续，并且满足info目录下的最小连续长度。
另外`0=`和`1=`表示不同socket上的资源限制。

### 为cpu绑定默认分配策略

将cpu号以掩码的方式写入rdt_group目录下的cpus文件，或者以范围的方式写入cpus_list。

```bash
echo 0-63 >  /sys/fs/resctrl/demo_rdt_group/cpus_list
```

相比于修改前新增的cpu如果已绑定到其他非根目录rdt_group会清除其他rdt_group中cpu的对应bit；被释放的cpu会返回到根rdt_group。并且写入后，当前rdt_group下的mon_groups目录中的监控组的cpus均被清零。

### 为task绑定分配策略

task被创建那一刻会加入根目录的rdt_group，所有的task只能属于最多一个rdt_group，当加入一个非默认的rdt_group时会为task绑定分配策略。

```bash
echo pid > /sys/fs/resctrl/demo_rdt_group/tasks
```

### 创建监控组

每一个rdt_group目录下都有mon_data存放监控事件值，resctrl提供了更细化的监控组策略，可以在rdt_group的mon_groups目录下新建新的监控组。

```bash
mkdir /sys/fs/resctrl/demo_rdt_group/mon_groups/demo_mon_grp
```

可以为cpu和task绑定监控组。只需要修改监控组的cpus_list和tasks文件。

```bash
echo 0-31 >  /sys/fs/resctrl/demo_rdt_group/mon_groups/demo_mon_grp/cpus_list
echo pid > /sys/fs/resctrl/demo_rdt_group/mon_groups/demo_mon_grp/tasks
```

需要注意，cpus_list必须是对应的资源组cpus_list的子集。pid同样。否则会写入失败。

### 策略的监控的实施规则

基本上关于resctrl的操作就只有以上内容，但是还是给人一种混乱的感觉，通过一通设置有可能还是搞不清楚task会收到怎样的资源限制以及监控组累积的数据包含哪些cpu或者是task的事件。这一部分我将在下一篇关于《resctrl的内核实现机制》中文章中进行说明。

## Reference

1. [User Interface for Resource Control feature](https://www.kernel.org/doc/html/v5.4/x86/resctrl_ui.html)
