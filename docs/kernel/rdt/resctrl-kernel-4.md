<!-- #! https://zhuanlan.zhihu.com/p/647041234 -->
# Resctrl内核实现（四）schemata和cpus
## 前言

在Resctrl中可编程的文件主要有两个，schemata指定资源分配策略，cpus则为cpu绑定`default_closid`和`default_rmid`。

## 修改cpus文件

cpus文件按照所处目录的类型不同有两种操作，当所处目录类型为CTRL-MON group时会调用`cpus_ctrl_write`，目录类型为MON group时会调用`cpus_mon_write`。通过对源码的分析，处理的过程包括：

1. 进行权限检查
2. 重设cpu的默认closid和rmid的

修改cpus会产生两种变化，一种是被drop掉的cpu，一种是新增的cpu。所谓drop就是相比于原cpus释放的cpu，新增则是相比于原cpus迁移来的cpu。

对于CTRL-MON group来说：

1. drop掉的cpu需要还给默认CTRL-MON group（因此默认CTRL-MON group无法进行drop cpu）；
2. 对于新增的cpu需要更新这些cpu的默认closid和rmid；对于这些新增cpu的原本所属的CTRL-MON group需要移除这些cpu；
3. 对于CTRL-MON group下的MON group需要**清空cpu_mask**，MON group清空掉的cpu需要更新对应默认closid和rmid为parent CTRL-MON group。

对于MON group来说：

1. 首先会检查新增的cpu是否属于parent CTRL-MON group，MON group不允许添加parent CTRL-MON group中不存在的cpu；
2. 检查是否有drop cpu，这些cpu需要放入到parent CTRL-MON group中，并且更新这些cpu的closid和rmid为parent CTRL-MON group；
3. 对于新增的cpu，如果是从其他MON group移入的需要在其他MON group中移除这些cpu，并且更新新增的cpu的默认closid和rmid为当前MON group。

```c
rdtgroup_cpus_write
    -->cpus_ctrl_write // ctrl group write cpus
        -->cpumask_andnot(tmpmask, &rdtgrp->cpu_mask, newmask)
            -->cpumask_or(&rdtgroup_default.cpu_mask, &rdtgroup_default.cpu_mask, tmpmask);
            -->update_closid_rmid // 检查是否有从控制组中释放的cpu，需要归还给默认控制组，将这些cpu的默认closid和rmid改为默认控制组
           -->cpumask_andnot(tmpmask, newmask, &rdtgrp->cpu_mask)
            -->list_for_each_entry(r, &rdt_all_groups, rdtgroup_list) // 遍历所有的rdtgroup，将新增的cpu从原本的group中移除
                -->cpumask_and(tmpmask1, &r->cpu_mask, tmpmask)
                -->cpumask_rdtgrp_clear(r, tmpmask1);
            -->update_closid_rmid(tmpmask, rdtgrp) // 更新这些cpu的closid和rmid
        -->cpumask_copy(&rdtgrp->cpu_mask, newmask); // 更新group的cpu_mask
        -->list_for_each_entry(crgrp, head, mon.crdtgrp_list)
            -->cpumask_and(tmpmask, &rdtgrp->cpu_mask, &crgrp->cpu_mask); // 对于mon group，对于child group丢失的cpu更新rmid和closid，清空child group的cpu_mask
            -->update_closid_rmid(tmpmask, rdtgrp);
            -->cpumask_clear(&crgrp->cpu_mask);

    -->cpus_mon_write(rdtgrp, newmask, tmpmask) // mon group write cpus
        -->cpumask_andnot(tmpmask, newmask, &prgrp->cpu_mask) // 检查cpus是否属于parent ctrl group,不允许添加一个parent group不存在的cpu
        -->cpumask_andnot(tmpmask, &rdtgrp->cpu_mask, newmask) // 检查是否有减少的cpu，将drop的cpu添加到父cpus中，更新这些cpu的closid和rmid
        -->cpumask_andnot(tmpmask, newmask, &rdtgrp->cpu_mask) // 对新增的cpu 如果属于其他的mon group，将这些cpu进行移除，并更新这些cpu的默认closid和rmid
        -->cpumask_copy(&rdtgrp->cpu_mask, newmask) // 更新cpu mask
```

## 修改schedmata

`rdtgroup_schemata_write`会对输入进行解析，按照不同的资源分别通过IPI的方式通知cpu调用`rdt_ctrl_update`进行更新，修改closid对应的cbm（cache bit mask）以及delay。

```c
rdtgroup_schemata_write
    -->rdtgroup_parse_resource
    -->for_each_alloc_enabled_rdt_resource(r)
        -->update_domains(r, rdtgrp->closid)
            -->smp_call_function_many(cpu_mask, rdt_ctrl_update, &msr_param, 1) // 通过IPI的方式通知closid涉及到的cpu更新资源控制
                
rdt_ctrl_update
    -->get_domain_from_cpu(cpu, r)
    -->msr_update(d, m, r); //msr_update 是一个函数指针 不同的资源有不同的更新函数
    -->mba_wrmsr or cat_wrmsr

static void
mba_wrmsr(struct rdt_domain *d, struct msr_param *m, struct rdt_resource *r)
{
    unsigned int i;

    /*  Write the delay values for mba. */
    for (i = m->low; i < m->high; i++)
        wrmsrl(r->msr_base + i, delay_bw_map(d->ctrl_val[i], r));
}

static void
cat_wrmsr(struct rdt_domain *d, struct msr_param *m, struct rdt_resource *r)
{
    unsigned int i;

    for (i = m->low; i < m->high; i++)
        wrmsrl(r->msr_base + cbm_idx(r, i), d->ctrl_val[i]);
}
```

在此场景下，m->low=closid，m->high=m->low+1，因此只会更新closid对应的资源分配机制。更新资源会按照closid，将控制的值（对于cache是cache bit mask，对于mb是延迟）写入资源控制的msr寄存器。
