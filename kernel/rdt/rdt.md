# RDT技术背景

- 全称 Resource Director Technolo，负责LLC（Last level cache）以及MB（Memory Bandwidth）内存带宽的分配和监控
- 内核版本支持：resctrl文件系统支持需要kernel version 4.10 以上
- 主要功能：
   - CAT（Cache Allocation Technology）：LLC的cache分配
   - CMT（Cache Monitor Technology）：LLC的cache使用情况监控
   - MBA（Memory Bandwidth Allocation）：内存带宽分配技术
   - MBM（Memory Bandwidth Monitor）：内存带宽监控技术
   - CDP（Code&Data Prioritization）：细化的Code Cache和Data Cache的分配
- 主要作用：在混合部署场景，cgroup提供了粗粒度的CPU、内存、IO等资源的隔离和分配，但是对于每个CPU，这些核心会独占自己的L1 Cache、L2 Cache，对于LLC则是共享的，在每个core的负载很高时就会产生对LLC的竞争，因此需要提供细粒度的LLC资源的分配。
  
# 实现机制

## 寄存器支持

RDT技术需要MSR（Model Specific Register）寄存器的支持，主要包括三个寄存器IA32_PQR_ASSOC、IA32_QM_EVTSEL和IA32_QM_CTR。

### IA32_PQR_ASSOC

![IA32_PQR_ASSOC寄存器](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1686125925668-58729ca1-2f89-4dcd-a41f-5a97acaa00ba.png#clientId=u222eb239-c318-4&from=paste&height=195&id=u514d0a78&originHeight=175&originWidth=641&originalType=binary&ratio=2&rotation=0&showTitle=true&size=34320&status=done&style=none&taskId=uaf735159-3718-4687-876e-d4c12e3eac8&title=IA32_PQR_ASSOC%E5%AF%84%E5%AD%98%E5%99%A8&width=713.5 "IA32_PQR_ASSOC寄存器")

负责指示当前线程所属的COS以及RMID，COS（Class of service）是一组资源控制策略，高32bit用于指示所属的策略组，RMID（Resource Monitor ID）则是监控组ID，指示当前线程所属的监控组。

### IA32_QM_EVTSEL

# ![IA32_QM_EVTSEL寄存器](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1686125994214-a4071ad2-e2b2-4707-8dd7-938ef7458307.png#clientId=u222eb239-c318-4&from=paste&height=274&id=ued53ac10&originHeight=250&originWidth=634&originalType=binary&ratio=2&rotation=0&showTitle=true&size=41920&status=done&style=none&taskId=u409991f7-9bc7-4612-8ce7-3ee46e278e8&title=IA32_QM_EVTSEL%E5%AF%84%E5%AD%98%E5%99%A8&width=694 "IA32_QM_EVTSEL寄存器")

是一个查询寄存器，在记录了查询的监控组的RMID以及想要查询的event以后，通过IA32_QM_CTR寄存器获取数据。
支持的查询事件包括：
L3 occupancy：监控组对L3cache的占有率
Total memory bandwidth：全部的内存带宽（local的内存带宽+remote内存带宽）
Local memory bandwidth：本地的内存带宽

### IA32_QM_CTR

![IA32_QM_CTR寄存器](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1686126059300-36652f6f-746b-48e9-abe6-6b42bd278c9b.png#clientId=u222eb239-c318-4&from=paste&height=193&id=u59d0fd1e&originHeight=170&originWidth=636&originalType=binary&ratio=2&rotation=0&showTitle=true&size=34527&status=done&style=none&taskId=ufc4fa97c-8b89-488a-8a03-39b7470a06c&title=IA32_QM_CTR%E5%AF%84%E5%AD%98%E5%99%A8&width=723 "IA32_QM_CTR寄存器")

当在IA32_QM_EVTSEL寄存器中写入了查询的事件以及查询的对象，IA32_QM_CTR中会存放查询的结果。

## 软件支持

1. CMT和MBM机制

硬件背景：CMT和MBM主要是一个监控机制，CMT和MBM能力通过三个寄存器实现，在IA32_PQR_ASSOC寄存器中写入一个RMID，对应的事件发生时相应的数据被统计到RMID对应的监控组。在IA32_QM_EVTSEL寄存器中写入想要查询的RMID以及事件ID后，可以从IA32_QM_CTR寄存器中读取当前数据。

软件实现：在硬件能力的基础上，通过软件设计实现RMID的分配机制，将监控值累计到不同的监控组。具体的分配机制可以自行拟定，需要注意的点有两个：
1. 在每一个core上都包含这三个寄存器，用于记录每一个core上发生的事件。
2. 不同socket上的rmid是独立的，不同socket上相同的RMID对应的累积数据会存在不同的地方
因此假设想要统计整个系统的一个情况，只需要分配一个RMID，让所有的core上的IA32_PQR_ASSOC的RMID相同，此时系统内所有的事件都可以通过一个RMID查询。
如果想要统计每个core上的情况，只需要为每个core分配不同的RMID，并写入IA32_PQR_ASSOC MSR寄存器。
如果想要统计当个task的情况，首先需要为task分配一个单独的RMID，并在task换入时将core上的RMID写入task绑定的RMID。

在task被CPU换入执行时，当前task对应的RMID会被写入IA32_PQR_ASSOC寄存器。当发生cache的换入时，如果cache line的tag和rmid对应，则cache的占用量会增加。如果cache_line被换出，cache line的tag对应的RMID的cache占用量会下降。关于MBM，也是同样，信息会统计到RMID对应的监控组。

1. CAT和MBA机制

当task切换时，当前task的CLOSID被写入IA32_PQR_ASSOC，通过CLOSID在IA32_L3_MASK_n数组上索引对应的CBM（cache bit mask），CBM描述了CLOSID对应的cache资源的使用量。MBA同样按照CLOSID在IA32_L2_QoS_Ext_BW_Thrtl_n数组上查找对应的delay。此时当前物理线程只能够访问特定区域的cache，并且访存会存在特定的延迟。

![CAT和MBA机制](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1686292928830-d0995f7d-01c7-49a1-a76a-d78202591333.png#clientId=u6d9dc58b-3940-4&from=paste&height=521&id=u02d89730&originHeight=553&originWidth=676&originalType=binary&ratio=2&rotation=0&showTitle=true&size=41624&status=done&style=none&taskId=u49c08bdc-659b-4a65-bebe-f1efd9038c5&title=CAT%E5%92%8CMBA%E6%9C%BA%E5%88%B6&width=637 "CAT和MBA机制")

# RDT的使用方法
## resctrl接口
在resctrl内包括两类策略管理机制：

1. **COS or CLOS（Class of service）**：配置CAT和MBA相关的策略
2. Monitor Groups：配置监控相关的策略
## 查看是否支持RDT技术
通过查看/proc/cpuinfo文件，检查cpu的flags是否包含对应的数据

| Feature | flag |
| --- | --- |
| RDT (Resource Director Technology) Allocation | "rdt_a" |
| CAT (Cache Allocation Technology) | "cat_l3", "cat_l2" |
| CDP (Code and Data Prioritization) | "cdp_l3", "cdp_l2" |
| CQM (Cache QoS Monitoring) | "cqm_llc", "cqm_occup_llc" |
| MBM (Memory Bandwidth Monitoring) | "cqm_mbm_total", "cqm_mbm_local" |
| MBA (Memory Bandwidth Allocation) | "mba" |
| SMBA (Slow Memory Bandwidth Allocation) | "" |
| BMEC (Bandwidth Monitoring Event Configuration) | "" |

## 启用RDT
通过mount挂在resctrl文件系统
```bash
mount -t resctrl resctrl [-o cdp[,cdpl2][,mba_MBps]] /sys/fs/resctrl
```

1. cdp：在L3 cache中开启code and data prioriization
2. cdpl2：在L2 cache中开启code and data prioriization
3. mba_MBps：启用MBA软件控制器，在MBps中指定MBA带宽

## Info目录
```bash
info
├── L3
│   ├── bit_usage
│   ├── cbm_mask
│   ├── min_cbm_bits
│   ├── num_closids
│   └── shareable_bits
├── L3_MON
│   ├── max_threshold_occupancy
│   ├── mon_features
│   └── num_rmids
├── last_cmd_status
└── MB
    ├── bandwidth_gran
    ├── delay_linear
    ├── min_bandwidth
    ├── num_closids
    └── thread_throttle_mode
```

每一个子目录包含一种可以被分配的资源。
Cache L2、L3包含一下和分配相关的文件：

1. num_closids：该资源的有效CLOSID，系统会使用所有启用的资源中有效CLOSID最小数量作为上限。
2. cbm_mask：表示有效资源的位掩码
3. min_cbm_bits：当写mask时，最小的连续比特数
4. shareable_bits：和其他执行实体共享的资源的位掩码。用户可以使用这个文件去设置独占的缓存分区。
5. bit_usage：一组掩码，表示资源实例如何被使用。一些标志位的含义如下
   1. “0”：表示资源未被使用，当资源已经被分配但并且发现该位说明资源被浪费了。
   2. “H”：对应的区域目前只被硬件使用，但是也可以软件使用。如果资源被设置了shareable_bits，但是不是所有的bits出现在控制组的schematas中，该bit出现在shareable_bits但是又没有控制组使用责备标记为“H”
   3. “X”：对应的区域可以被共享，同时被硬件和软件使用。这些位表示出现在shareable_bits，又被resource group分配了。
   4. “S”：表示区域被软件使用了，同时可以被共享。
   5. “E”：表示资源被一个控制组独占，不允许共享。
   6. “P”：对应的区域被pseudo-locked了，不允许共享。

内存带宽（Memory Bandwidth， MB）子目录包含以下和分配相关的文件：

1. min_bandwidth：用户可以请求的最小的内存带宽百分比
2. bandwidth_gran：内存带宽百分比的粒度，被分配的内存比例被被四舍五入到下一个粒度对齐的位置，有效的带宽控制位置为：min_bandwidth+N*bandwidth_gran
3. delay_linear：表示延迟刻度是线性的还是非线性的
4. thread_throttle_mode：表示在Intel系统上，运行在物理核心的线程上的task在请求不同的内存带宽占比时是如何被限流的。
   1. “max”：所有的线程使用最小的百分比，意思是两个线程如果一高一低，都使用低百分比。
   2. “per-thread”：带宽百分比直接被应用在运行在核心上的线程。

如果RDT监控是可用的，会有一个L3_MON子目录，包含以下文件：

1. num_rmids：有效RMID的数量，也是能够创建的CTRL_MON+MON groups的上限。
2. mon_features：支持的监控事件
   
```bash
llc_occupancy
mbm_total_bytes
mbm_local_bytes
```

如果支持Bandwidth Monitoring Event Configuration (BMEC)，带宽监控事件配置，那么带宽事件是可以配置的，支持的事件如下：

```bash
llc_occupancy
mbm_total_bytes
mbm_total_bytes_config
mbm_local_bytes
mbm_local_bytes_config
```

mbm_total_bytes_config和mbm_local_bytes_config两个为可配置事件，当修改任意一个事件配置时，mbm_total_bytes和mbm_local_bytes都会被清零，下一次读取时会报告Unavailable，后续的read将会返回有效值。
可配置的事件如下，默认情况下mbm_total_bytes被设置为0x7f，也就是采集所有的事件，mbm_local_bytes被设置为0x1f，表示只采集访问local的事件。可以通过配置的方式，只采集读事件、写事件等等。

| Bits | Description |
| --- | --- |
| 6 | Dirty Victims from the QOS domain to all types of memory |
| 5 | Reads to slow memory in the non-local NUMA domain |
| 4 | Reads to slow memory in the local NUMA domain |
| 3 | Non-temporal writes to non-local NUMA domain |
| 2 | Non-temporal writes to local NUMA domain |
| 1 | Reads to memory in the non-local NUMA domain |
| 0 | Reads to memory in the local NUMA domain |

3. max_threshold_occupancy：读写该文件，提供一个最大字节数，一个之前使用的LLC_occupancy计数器可以在达到该值时被考虑重用。

最后info目录下包含一个last_cmd_status，包含上一条执行命令的执行结果，ok表示成功或者其他的错误信息。

## 资源分配和监控组
在resctrl文件系统中，控制组通过目录来表示。默认的控制组就是根目录，在mount后建立，拥有所有的tasks和cpus，使用所有的资源。
在支持资源控制的OS上，可以在根目录创建新的子目录，表示信息的CTRL_MON group。在一个支持资源监控的OS上，可以在根目录控制组或者其他控制住组的mon_groups下新建监控组，监控当前控制组中的tasks。这些监控组被称为MON group。
移除一个控制组会将其task和cpu都移到其parent控制组中，移除CTRL_MON group会自动移除MON group
不管是CTRL_MON group还是MON group都包含以下文件：

1. tasks：读该文件会返回所有属于该组的task列表。写该文件时会将task添加到该组，如果group是一个CTRL_MON，该task属于的CTRL_MON或者MON都会移除该task；如果该组是MON，该task必须已经属于了MON所属的CTRL_MON，并且该task过去所在MON（可能有）也要移除该task。总结一下每个task必须属于一个CTRL_MON，并且至多属于该CTRL_MON group的MON group。
2. cpus：读取该文件显示该group拥有的逻辑cpu的掩码。写该文件可以从该group中添加或者移除cpu。和tasks文件一样，MON group只能包含上一级CTRL_MON group包含的cpu。
3. cpus_list：和cpus一样，只是使用范围表示

CTRL_MON group包含以下文件：

1. schemata：可用的资源列表
2. size：和schemata一样，但是显示的是字节数，而不是掩码。
3. mode：表明控制组是否共享其分配的资源。“shareable”表示允许共享。在将cache 伪锁区域的schemata写入控制组的schemata文件前，将“pseudo-locksetup”写入mode文件，会创建一个cache的伪锁区域，但成功创建伪锁区域后将自动将mode修改为“pseudo-locked”

MON group包含以下文件：

1. mon_data：是一个目录，对每一个L3的domain都有一个子目录，每个domain的目录下包含对应RDT事件的累积值，CTRL_MON下的mon_data，会包含tasks所有的总和。
```bash
/sys/fs/resctrl/test_group1/mon_groups/mon1/mon_data/
├── mon_L3_00
│   ├── llc_occupancy
│   ├── mbm_local_bytes
│   └── mbm_total_bytes
└── mon_L3_01
    ├── llc_occupancy
    ├── mbm_local_bytes
    └── mbm_total_bytes
```

### 资源分配规则

1. 如果一个task属于一个非默认的CTRL_MON group，则使用该组的资源列表
2. 如果task属于默认的CTRL_MON group，但是task当前所在的cpu属于某个非默认的CTRL_MON group，则使用对应组的资源列表
3. 否则，使用默认的CTRL_MON group的资源列表
   
### 资源监控规则

1. 如果一个task属于某个MON group或者某个非默认CTRL_MON group，则RDT事件记录在该group中
2. 如果task属于默认的CTRL_MON group，但是当前运行的cpu属于某个非默认CTRL_MON group，此时记录在对应的CTRL_MON group中。
3. 否则，记录在默认的CTRL_MON group中

## Cache占用的监控和控制

1. 当将一个task从一个组移动到另一个组时，只会影响到新的cache line的分配，因此当移动以后旧的组的cache占用并不会随着task的移动减少，因为这些cache还在使用当中，随着系统运行，旧的cache line会被换出和被新的group重用，然后统计到新的group中。
2. 硬件使用CLOSID和RMID识别一个特定的控制组和监控组。CLOSID和RMID会受到硬件的限制，所以控制组和监控组的个数有限。
   
### max_threshold_occupancy

RMID释放后可能无法立刻有效，因为RMID可能还用于标记某些cache line，这些RMID会放在limbo链表中，等待cache占用下降后再来检查能否释放，如果某个时刻系统有很多的limbo RMID，但是还不可用，此时在创建目录时就会看到EBUSY。max_threshold_occupancy是一个可配置的值，表示一个RMID可以释放的cache line水位，低于该水位时表示RMID可以再次被分配。

### schemata

每一行描述一种资源，包括资源名和资源在实例上的分配情况。

### Cache IDs

1. Cache架构：现代系统一般每个socket有一个L3 cache，每个core上的超线程共享L2 cache，但是这不是一个架构的前提和限制。也可以在每个socket上有多个L3 cache，每个core上有多个L2 cache，因此不使用socket或者core来定义一组共享资源的cpu集合，而是使用Cache ID来描述，对于给定的cache level，都有一个独一无二的数字来表示。
2. 可以通过/sys/devices/system/cpu/cpu*/cache/index*/id来查看每个逻辑cpu的ID。
   
### Cache Bit Masks(CBM)

1. 对于每个cache资源，通过bit mask对cache的占比进行表示，每个cache的mask位数可能不同，可以通过info/{resource}/cbm_mask查看。
2. 硬件要求，cache bit mask的“1”必须连续。0x5=0101b 是不允许的。
   
## 内存带宽的监控和控制

对于内存带宽资源，用户通过指示在全部内存带宽的占比来控制该资源。
对于每个cpu模型的最小的带宽占比是一个预定义的值，可以通过info/MB/min_bandwidth查看。分配的带宽粒度通过info/MB/bandwidth_gran查看。有效的带宽比例为：min_bw+ N * bw_gran，中间值被四舍五入到最近的有效比例中。
带宽限流在一些intel SKUs上是一种core特定的机制，在一个core上的两个线程上使用高低两种带宽，可能会导致两个线程都被限流。具体的策略可以看info/MB/thread_throttle_mode文件。

1. 用户在增加了百分比后看不到实际带宽的增加

当聚合的L2的外部带宽超过L3外部带宽时可能出现这种情况。假设一个处理器有24个cores，每个core上的L2外部带宽为10GBps，L2聚合带宽为240GBps，L3外部带宽为100GBps。假设工作负载有20个线程，占有50%的带宽，因此每个线程消耗5GBps的贷款，总共消耗100GBps的带宽，此时虽然带宽占比50%远小于100%，但是增加core的带宽比例并不会增加实际的带宽总量，因为L3带宽已经拉满。

1. 相同的带宽比例可能实际带宽总量不同

因为带宽限制是对core来说的，假设对带宽比例为10%，对于只有一个线程的core和有4个线程的core，其带宽上限分别是10%和40%，是不一样的。对于使用不同cores数量的rdtgroup来说，相同的带宽比例也会造成实际带宽的不同。

### L3 schemata file details
未开启CDP时，format如下：
```bash
L3:<cache_id0>=<cbm>;<cache_id1>=<cbm>;...
```
开启CDP时的format如下：
```bash
L3DATA:<cache_id0>=<cbm>;<cache_id1>=<cbm>;...
L3CODE:<cache_id0>=<cbm>;<cache_id1>=<cbm>;...
```
### L2 schemata file format
```bash
L2:<cache_id0>=<cbm>;<cache_id1>=<cbm>;...
or
L2DATA:<cache_id0>=<cbm>;<cache_id1>=<cbm>;... 
L2CODE:<cache_id0>=<cbm>;<cache_id1>=<cbm>;...
```
### 内存带宽分配
默认模式使用百分比，可以指定MBps
```bash
MB:<cache_id0>=bandwidth0;<cache_id1>=bandwidth1;...
MB:<cache_id0>=bw_MBps0;<cache_id1>=bw_MBps1;...
```
### 慢内存带宽分配（Slow Memory Bandwidth Allocation，SMBA)
AMD硬件支持慢速内存带宽分配，CXL.memory是唯一支持的“慢速”内存设备。如果有多个这样的设备，限流的逻辑会聚集这些慢速设备在一起，然后进行限制。SMBA和这些设备无关，配置SMBA在没有慢速内存的系统上不会有任何影响。
慢速内存的带宽域是L3 cache。schemata格式为：

```bash
SMBA:<cache_id0>=bandwidth0;<cache_id1>=bandwidth1;...
```
### 读写schemata文件

1. 在intel平台下

读schemata会显示所有资源在所有domain上的状态，写文件只需要指定想要修改的值
```bash
# cat schemata
L3DATA:0=fffff;1=fffff;2=fffff;3=fffff
L3CODE:0=fffff;1=fffff;2=fffff;3=fffff
# echo "L3DATA:2=3c0;" > schemata
# cat schemata
L3DATA:0=fffff;1=fffff;2=3c0;3=fffff
L3CODE:0=fffff;1=fffff;2=fffff;3=fffff
```

2. AMD平台下

AMD平台下的MB为value*1/8 GBps，因此指定domain1上的带宽总量为2GBps需要写入value=16.
```bash
# cat schemata
  MB:0=2048;1=2048;2=2048;3=2048
  L3:0=ffff;1=ffff;2=ffff;3=ffff

# echo "MB:1=16" > schemata
# cat schemata
  MB:0=2048;1=  16;2=2048;3=2048
  L3:0=ffff;1=ffff;2=ffff;3=ffff
```

3. AMD系统下并且支持SMBA特性

分配8GBps的限制在第一个cache id上
```bash
# cat schemata
  SMBA:0=2048;1=2048;2=2048;3=2048
    MB:0=2048;1=2048;2=2048;3=2048
    L3:0=ffff;1=ffff;2=ffff;3=ffff

# echo "SMBA:1=64" > schemata
# cat schemata
  SMBA:0=2048;1=  64;2=2048;3=2048
    MB:0=2048;1=2048;2=2048;3=2048
    L3:0=ffff;1=ffff;2=ffff;3=ffff
```
## Cache pesudo lock

CAT允许用户指定使用的cache空间。Cache pesudo lock基于这样一个现象：当cache命中时，CPU可以读写已经分配好的但是是自己分配区域以外的cache。当cache被上了pesudo lock时，数据可以被预加载到保留的cache空间中，就能保证cache命中。这样一个应用就可以通过这种方式映射自己的虚拟地址空间到内存上，并且这块区域的内存读写延迟非常低。
创建一个cache的pesudo lock区域的流程如下：

1. 创建一个CLOS CLOSNEW，并且CBM使用的cache不和其他已有的CLOS冲突，并且未来也不会出现重合
2. 创建一个连续的内存区域和cache区域大小相同
3. 清空cache，禁用硬件预取，禁止抢占
4. 让CLOS CLOSNEW开始活动，访问分配的内存装载入cache
5. 让之前的CLOS开始活动
6. 此时CLOSNEW可以被释放了，只要未来cache不与其他的CLOS重合，就能保证这块内存区域始终hit
7. 暴露这块连续的内存作为一个字符设备给用户

Cache pesudo lock增加了数据保留在cache中的概率，通过配置CLOS以及控制应用行为提高了cache命中率，但是pesudo lock并不保证数据一定在cache中，一些指令仍然可以让这些被“锁住”的cache line被换出。
因此需要要求应用运行在和pesudo lock区域关联的cores上，这样才能让cache驻留住。

总结伪锁区域的建立步骤：
1. 第一个阶段：系统管理员分配一块cache并进行pesudo lock（尽可能不让cache line换出），然后此时分配一个等大的内存区域，装载到cahe区域中，并暴露为一个字符设备
2. 第二个阶段：用户态应用程序映射被锁住的内存区域到自己的虚拟地址空间进行访问

### Cache pesudo lock接口
通过resctrl文件系统

1. 创建一个资源组：通过在/sys/fs/resctrl目录下新建目录
2. 修改新的资源组的mode为“pseudo-locksetup”：将“pseudo-locketup”写入mode文件
3. 写pesudo lock区域的schemata到schemata文件，所有在schemata文件中的bits应该都是未使用的(根据/sys/fs/resctrl/info/L3/bit_usage判断)

如果成功，mode文件内会被写入“pseudo-locked”，一个和资源组名字相同的新的字符设备将会出现在/dev/pseudo_lock目录下，这个设备将可以mmap到用户空间。

# 参考

1. [Kernel 4.14+ Intel RDT / AMD PQOS配置](https://zhuanlan.zhihu.com/p/92125001)
2. [User Interface for Resource Control feature — The Linux Kernel documentation](https://docs.kernel.org/arch/x86/resctrl.html?highlight=resctrl)
3. [深入浅出Intel RDT技术（二）— 软件实现](https://ata.alibaba-inc.com/articles/87547?spm=ata.23639746.0.0.32673565o0XDKy#vhwBGKkn)
4. [Introduction to Memory Bandwidth Monitoring in the Intel® Xeon®](https://www.intel.cn/content/www/cn/zh/developer/articles/technical/introduction-to-memory-bandwidth-monitoring.html)
