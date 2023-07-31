#! https://zhuanlan.zhihu.com/p/647036878

# Intel-RDT 技术浅析

本文适合于想要了解RDT技术的人阅读，会涉及到RDT技术的硬件机制，需要对CPU的socket、core、thread等概念有一定的了解。

## RDT技术简介

RDT技术全称 Resource Director Technology，RDT技术提供了LLC（Last level cache）以及MB（Memory Bandwidth）内存带宽的分配和监控能力。

RDT的主要功能有以下几个：

1. CAT（Cache Allocation Technology）：LLC的cache分配
2. CMT（Cache Monitor Technology）：LLC的cache使用情况监控
3. MBA（Memory Bandwidth Allocation）：内存带宽分配技术
4. MBM（Memory Bandwidth Monitor）：内存带宽监控技术
5. CDP（Code&Data Prioritization）：细化的Code Cache和Data Cache的分配

在混合部署场景，cgroup提供了粗粒度的CPU、内存、IO等资源的隔离和分配，但是对于每个CPU，这些核心会独占自己的L1 Cache、L2 Cache，对于LLC则是共享的，cgroup无法解决llc的竞争问题，因此需要提供细粒度的LLC资源的分配和监控。

## Intel RDT

### 硬件能力

RDT技术提供了MSR（Model Specific Register）寄存器作为编程接口，主要包括三个寄存器IA32_PQR_ASSOC、IA32_QM_EVTSEL和IA32_QM_CTR。

#### IA32_PQR_ASSOC

![IA32_PQR_ASSOC寄存器](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1686125925668-58729ca1-2f89-4dcd-a41f-5a97acaa00ba.png#clientId=u222eb239-c318-4&from=paste&height=195&id=u514d0a78&originHeight=175&originWidth=641&originalType=binary&ratio=2&rotation=0&showTitle=true&size=34320&status=done&style=none&taskId=uaf735159-3718-4687-876e-d4c12e3eac8&title=IA32_PQR_ASSOC%E5%AF%84%E5%AD%98%E5%99%A8&width=713.5 "IA32_PQR_ASSOC寄存器")

负责指示当前物理线程所属的COS以及RMID，COS（Class of service）是一组资源控制策略，高32bit用于指示所属的策略组，RMID（Resource Monitor ID）则是监控组ID，指示当前物理线程所属的监控组。

#### IA32_QM_EVTSEL

![IA32_QM_EVTSEL寄存器](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1686125994214-a4071ad2-e2b2-4707-8dd7-938ef7458307.png#clientId=u222eb239-c318-4&from=paste&height=274&id=ued53ac10&originHeight=250&originWidth=634&originalType=binary&ratio=2&rotation=0&showTitle=true&size=41920&status=done&style=none&taskId=u409991f7-9bc7-4612-8ce7-3ee46e278e8&title=IA32_QM_EVTSEL%E5%AF%84%E5%AD%98%E5%99%A8&width=694 "IA32_QM_EVTSEL寄存器")
是一个查询寄存器，在记录了查询的监控组的RMID以及想要查询的event以后，通过IA32_QM_CTR寄存器获取数据。
支持的查询事件包括：

- L3 occupancy：Last-level-cache的占有量
- Total memory bandwidth：全部的内存带宽（local的内存带宽+remote内存带宽）
- Local memory bandwidth：本地的内存带宽

#### IA32_QM_CTR

![IA32_QM_CTR寄存器](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1686126059300-36652f6f-746b-48e9-abe6-6b42bd278c9b.png#clientId=u222eb239-c318-4&from=paste&height=193&id=u59d0fd1e&originHeight=170&originWidth=636&originalType=binary&ratio=2&rotation=0&showTitle=true&size=34527&status=done&style=none&taskId=ufc4fa97c-8b89-488a-8a03-39b7470a06c&title=IA32_QM_CTR%E5%AF%84%E5%AD%98%E5%99%A8&width=723 "IA32_QM_CTR寄存器")
用于获取监控的事件值。

### CLOSID和RMID

Intel 提供了内存带宽和llc的分配和监控能力，其核心就在于**CLOSID（Class of Service ID）**和**RMID（Resource Monitor ID）**。每一个物理线程的IA32_PQR_ASSOC寄存器指示着当前线程访问资源的限制方式以及事件监控累积到的具体监控组。每一个core上的每一个物理线程（可能存在超线程情况）都包含一组RDT的MSR寄存器。

CLOSID是一组资源分配策略的索引，由硬件支持，当物理线程发起对llc的访问时通过CLOSID作为偏移在**IA32_L3_QoS_MASK_n**中找到CBM（Cap Bit Mask），CBM是位掩码，表示对llc的分配策略，按照CBM访问限定区域的cache。同时CLOSID也作为**IA32_L2_QoS_Ext_BW_Thrtl_n**的偏移得到Delay Value，Delay Value限定当前物理线程访问共享的LLC时的速率，通过增加对LLC的访问延迟来改变实际访问的内存带宽。可以参考下图：

![CAT和MBM实现](https://ata2-img.oss-cn-zhangjiakou.aliyuncs.com/neweditor/f14eeed9-8ec0-4f8a-8442-f3545d393b6a.png)

RMID则是用于标记当前物理线程上的事件应该统计到的具体监控组。

**Note**：在这里有几个需要注意的点：

1. IA32_PQR_ASSOC、IA32_QM_EVTSEL和IA32_QM_CTR三个寄存器是RDT技术的可编程接口寄存器，用于配置分配策略和读取监控事件，需要注意这组寄存器是**per-thread**的，因此可以在per-thread粒度上进行资源配置和监控。
2. IA32_L3_QoS_MASK_n和IA32_L2_QoS_Ext_BW_Thrtl_n是具体存储内存带宽和llc分配策略的寄存器组，是**per-socket**的，每个socket的多个core上的多个thread共享该组，很明显这两组寄存器的数量是有限的，因此**CLOSID的数量也是有限**的，如何分配CLOSID、如何切换thread的CLOSID这是上层软件需要关注的事情。
3. RMID与CLOSID类似，硬件不可能支持无限的监控组，因此**RMID的数量也是有限**的。
4. 第2条中提到，IA32_L3_QoS_MASK_n与IA32_L2_QoS_Ext_BW_Thrtl_n是per-socket的硬件寄存器，因此RMID和CLOSID自然也是per-socket的，**不同socket上的相同ID完全独立**。
5. 上文在描述内存带宽限制时提到了可编程的请求控制器，该控制器比较特殊，在**某些CPU上是per-core**的，因此在特定的CPU上同core上的不同thread即使具备不同的delay value，会按照两个thread中最大程度的内存带宽限制进行压制。当然**也有一些CPU支持per-thread的细粒度内存带宽限制**。

## RDT技术应用

RDT技术作为intel提供硬件特性，自然也存在一些相应的上层软件用于用户使用RDT能力，比如pqos、pcm等等。Linux内核在4.10版本也提供了resctrl文件系统用于操控RDT。关于resctrl的机制和实现后续再写。
