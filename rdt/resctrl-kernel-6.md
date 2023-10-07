#! https://zhuanlan.zhihu.com/p/647043798
# Resctrl内核实现（六）监控事件的记录

RDT出了提供资源的分配能力外，还提供了对llc和内存带宽等资源的监控能力，用于系统的争抢检测进行性能优化，在resctrl文件系统中监控的数据放在mon_data目录下。

## 监控事件的记录

内核代码主要在`/arch/x86/kernel/cpu/intel_rdt_monitor.c`。

监控组功能是实现对llc占用、total mb、local mb等指标进行统计。在kernel中是通过IPI的方式通知cpu。在对resctrl的文件进行读取时或者第一次创建监控组时会触发对指标的读取，单独进行指标读取时只会对当前文件相关的事件进行读取，第一次创建监控组时则会对所有事件进行读取。
rdt_domain是一组共享资源的cpu，`mon_event_read`会通过IPI的方式通知domain中的cpu去按照event id读取寄存器。

```c
void mon_event_read(struct rmid_read *rr, struct rdt_domain *d,
            struct rdtgroup *rdtgrp, int evtid, int first)
{
    /*
     * setup the parameters to send to the IPI to read the data.
     */
    rr->rgrp = rdtgrp;
    rr->evtid = evtid;
    rr->d = d;
    rr->val = 0;
    rr->first = first;

    smp_call_function_any(&d->cpu_mask, mon_event_count, rr, 1);
}

int rdtgroup_mondata_show(struct seq_file *m, void *arg)
{
    ...
    mon_event_read(&rr, d, rdtgrp, evtid, false);
    ...
}


static int mkdir_mondata_subdir(struct kernfs_node *parent_kn,
                struct rdt_domain *d,
                struct rdt_resource *r, struct rdtgroup *prgrp)
{
    list_for_each_entry(mevt, &r->evt_list, list) {
        priv.u.evtid = mevt->evtid;
        ret = mon_addfile(kn, mevt->name, priv.priv);
        if (ret)
            goto out_destroy;

        if (is_mbm_event(mevt->evtid))
            mon_event_read(&rr, d, prgrp, mevt->evtid, true);
    }
    
}
```

`mon_event_count`会判断统计的group类型，如果是MON group则从寄存器中读取累计值返回，如果是CTRL-MON group则在读取了累计值后，还会对下面的每一个MON group进行读取后进行加和。

```c
/*
 * This is called via IPI to read the CQM/MBM counters
 * on a domain.
 */
void mon_event_count(void *info)
{
    struct rdtgroup *rdtgrp, *entry;
    struct rmid_read *rr = info;
    struct list_head *head;
    u64 ret_val;

    rdtgrp = rr->rgrp;

    ret_val = __mon_event_count(rdtgrp->mon.rmid, rr);

    /*
     * For Ctrl groups read data from child monitor groups and
     * add them together. Count events which are read successfully.
     * Discard the rmid_read's reporting errors.
     */
    head = &rdtgrp->mon.crdtgrp_list;

    if (rdtgrp->type == RDTCTRL_GROUP) {
        list_for_each_entry(entry, head, mon.crdtgrp_list) {
            if (__mon_event_count(entry->mon.rmid, rr) == 0)
                ret_val = 0;
        }
    }

    /* Report error if none of rmid_reads are successful */
    if (ret_val)
        rr->val = ret_val;
}
```
