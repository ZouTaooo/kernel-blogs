# uncore

## uncore简介

硬件事件按照处于的位置划分为`core`、`offcore`和`uncore`事件，`core`事件指的是最小的逻辑核上的事件，比如`cycle`或者`instruciton`；由于每个物理核会有多个逻辑核，同一物理核上的不同逻辑核会有一些资源在共享，比如同一物理核的多个逻辑核访存时走的就是相同的通道，属于`offcore`事件；`uncore`是多个物理核之间共享的资源的事件，比如`LLC`，这些资源被划分到`uncore`。
`uncore`子系统的组件很多，包括：

* `CBox`(`caching agent`)
* `PCU`(`power controller unit`)
* `iMC`(`integrated memory controller`)
* `HA`(`home agent`)
* ......

## PMON

`uncore`的性能监控能力通过`per-component`的性能监控单元支持，这种组件内的监控单元也被称作`PMON`，每一个组件内的`PMON`包含一个或者多个`register`集合。`UBox`是一个例外，每个`PMON`单元提供了一个`unit-level`的控制寄存器去同步`UBox`中的多个计数器。

通过读取`local`计数器寄存器可以收集事件信息，每一个计数器寄存器都有一个配对的控制寄存器，通过改变`event select`/`umask`指定对什么事件计数以及如何计数，一些`unit`还提供了一些额外信息去`filter`监控事件。

需要注意的是，`uncore`监控的是`socket`级别的事件，`uncore`事件不受到内核调度的线程迁移的影响，因此建议设置Task的亲和性绑定到指定`socket`上，避免线程跨`socket`计数到不同的`uncore PMU`上。

`uncore`的计数寄存器和控制寄存器的可编程接口的地址空间有两种：

* `Cbo Units`、`PCU`以及`U-Box`的`PMON`寄存器通过`MSR`地址空间访问
* `HA`、`iMC`、`Intel QPI`、`R2PCIe`以及`R3QPI`单元通过`PCI device configuration`地址空间（`PCICFG`）访问

## uncore PMON的控制/计数寄存器

下面这个图展示了计数寄存器中`event`信息是如何流转以及存储的，以及控制寄存器如何进行`event`选择和过滤的。
![Alt text](pic/perfom-control-counter-block.png)

控制寄存器最主要的功能就是去配置监控的事件，通过设置`.ev_sel`和`.umask`字段实现对事件的选择。

配置了事件以后需要通知硬件控制器寄存器被修改，`.en`位必须被设置为`1`去开启计数。一旦`box`和全局级别的计数被使能，对应的数据寄存器就可以开始收集事件了。

`uncore`还有一些高级的特性：

* `.thresh`: 设置事件的阈值
* `.invert`: 设置统计规则是高于阈值还是低于阈值
* `.edge_det`: 统计事件变化次数

比如想要统计内存每次读写超过`128KB`的事件数，此时通过设置`.thresh`为非零数，只有到来的事件超过或者低于这个阈值计数才会加一。`.invert`和`.edge_det`发生在`.thresh`比较之后，`.invert`用于取反，表示小于。`.edge_det`统计事件状态变化的次数，比如从没有事件发生到事件触发的次数。

## Refreence

1. xeon-e5-2600-uncore-guide
2. [PERF EVENT 硬件篇续](https://ata.alibaba-inc.com/articles/104772)