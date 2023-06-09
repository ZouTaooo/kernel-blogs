# resctrl

resctrl是rdt机制的一个用户态接口，通过对rdt技术进行封装，提供了一套资源分配和监控机制的接口，方便用户进行使用。本文从resctrl的资源分配和监控的角度对内核源码实现进行了分析，参考的kernel版本为4.19.287。

## resctrl实现总览

resctrl中有两种group，控制组和监控组，控制组（CTRL group）进行资源分配，监控组（mon group）负责进行指标采集，而resctrl中将控制组和监控组进行了组合，被称作CTRL-MON group。resctrl根目录就是一个CTRL-MON group，这种组兼顾了资源分配规则和监控。

每个cpu在不同的时刻会运行不同task，而resctrl为了支持复杂的资源分配和监控逻辑，每个cpu不同的时刻会存在不同的状态。
下面这个结构体是per-cpu的，每个cpu都拥有`default_rmid`、`default_closid`以及`cur_rmid`、`cur_closid`。这个设计非常关键，正是这个结构体实现了resctrl的核心逻辑。

```c
struct intel_pqr_state {
    u32   cur_rmid;
    u32   cur_closid;
    u32   default_rmid;
    u32   default_closid;
};
```

我们知道RMID用于决定当前的cpu上发生事件时应该累积到哪个监控组去，而CLOSID让当前cpu去查询对应的CBM（cache bit mask），去访问规定范围的资源。`cur_rmid`和`cur_closid`就对应了当前cpu的IA32_PQR_ASSOC寄存器里的closid和rmid值。`cur_rmid`和`cur_closid`是会改变的，当cpu换入的task被指定了`cur_rmid`或者`cur_closid`时，就会更新对应的id，需要注意这里用到了或这个字，这是因为可以给task单独指定rmid而不指定closid；当cpu换入的task没有指定rmid和closid时cpu就会使用默认的rmid和closid。因此resctrl文件系统的逻辑就是在给cpu指定默认的rmid和closid，以及给task分配和释放固定的rmid和closid。

resctrl中的每个CTRL-MON group和MON group都被分配了一个CLOSID和RMID，每个cpu的bit都只能在一个CTRL-MON group有效，因此cpu的默认rmid和closid就是所属的CTRL-MON group的rmid和closid，由于CTRL-MON group下可以新建MON-group，当cpu被分配到当前CTRL-MON group中去时，cpu的默认rmid会更新为CTRL-MON group的rmid。

至于task的closid，当task被创建时会加入默认的CTRL-MON group，此时task是没有指定closid和rmid的，当task被写入一个新的CTRL-MON group时，此时task的closid和rmid指定为新的CTRL-MON group的closid和rmid。关于rmid，当task被写入到一个mon_groups目录下的MON group时，task的rmid会被更新。

在充分理解了以上closid和rmid的分配与使用规则后，就能够很好理解resctrl中task的资源使用和监控规则。

资源分配规则：

1. 当一个task加入一个非默认的CTRL-MON group时，此时task在运行中时，不管在哪个cpu上都受到非默认CTRL-MON group的schemata限制；
2. 如果一个task属于默认CTRL-MON group，也就是没有为task指定closid，此时如果运行的cpu属于一个非默认CTRL-MON group，按照非默认CTRL-MON group的schemata的限制访问资源。其实说白了也就是在哪个cpu上按照哪个cpu的默认规矩来；
3. 其他情况，按照默认CTRL-MON group的schemata访问资源。

监控规则：

1. 当一个task加入了一个mon_groups目录下的MON group，或者属于一个非默认的CTRL-MON group，此时关于task的rdt事件会报告到所属的MON-group中；
2. 如果一个task不属于某个MON group，同时不属于某个非默认的CTRL-MON group，此时按照运行的cpu进行划分，如果当前cpu属于非默认CTRL-MON group，则报告到非默认CTRL-MON group；
3. 其他情况，报告给默认CTRL-MON group的监控组。

resctrl文件系统操作和CLOSID以及RMID的分配关系如下：

1. 当挂载resctrl会指定一个默认的cosid和rmid进行初始化操作，此时每一个cpu的默认closid和rmid都会被更新，对应着默认控制组和对应的监控组。
2. 当创建一个新的CTRL-MON group时会分配一个新的cosid和rmid。
3. 当在mon_groups下创建新的MON group时也会分配一个closid和rmid，但是感觉这个closid的意义不明，因为在MON group中并不能修改schemata文件，制定资源分配规则。

对于挂在了resctrl的场景下，对task的操作与CLOSID以及RMID关系如下：

1. 当一个task被创建时此时task首先进入默认CTRL-MON group，此时task没有指定cosid和rmid；
2. 当task被移入一个非默认的CTRL-MON group时，task的cosid和rmid都被更新为对应的id，此时如果再将task写入到一个创建的MON group时，会重新指定task的rmid，当然task的写入会存在限制，任意一个task只能处于一个CTRL-MON group组，并且最多属于该CTRL-MON group的mon_groups目录下的一个MON group（也可以不属于，使用CTRL-MON默认的rmid）。

对于cpus文件的操作与CLOSID和RMID的关系如下：

1. 当修改控制组的cpus_list时，对应的cpu的默认cosid和rmid会更新为控制组的cosid和rmid。

## RDT内核实现

### closid和rmid管理

#### closid管理

rdt中的资源通过控制组进行分配，每个控制组对于每一类资源进行划分或者限制，每一个控制组用一个closid进行标识，在内核中由于closid的数量有限所以通过一个位图进行表示和管理。

```c
static int closid_free_map;
static int closid_free_map_len;

// 查看closid的有效范围
int closids_supported(void)
{
    return closid_free_map_len;
}

// 初始化cosid的位图和位图长度
static void closid_init(void)
{
    struct rdt_resource *r;
    int rdt_min_closid = 32;

    /* Compute rdt_min_closid across all resources */
    for_each_alloc_enabled_rdt_resource(r)
        rdt_min_closid = min(rdt_min_closid, r->num_closid);

    closid_free_map = BIT_MASK(rdt_min_closid) - 1;

    /* CLOSID 0 is always reserved for the default group */
    closid_free_map &= ~1;
    closid_free_map_len = rdt_min_closid;
}

// 分配一个closid
static int closid_alloc(void)
{
    u32 closid = ffs(closid_free_map);

    if (closid == 0)
        return -ENOSPC;
    closid--;
    closid_free_map &= ~(1 << closid);

    return closid;
}

// 释放closid
void closid_free(int closid)
{
    closid_free_map |= 1 << closid;
}

// 检查closid是否已分配
static bool closid_allocated(unsigned int closid)
{
    return (closid_free_map & (1 << closid)) == 0;
}

```

#### rmid管理

监控利用rmid标志一个监控组，进行累积的时候会按照rmid的值统计到对应的监控组中，rmid在内核中通过一个链表进行管理，每个rmid使用`rmid_entry`结构体表示，`busy`标志位表示该rmid正在使用中。

```c
struct rmid_entry {
    u32    rmid;
    int    busy;
    struct list_head  list;
};
```

但是由于RMID的数量有限，就会涉及到RMID的分配与释放过程。

##### alloc rmid

`alloc_rmid`分配一个rmid，会从链表中找出一个空闲rmid。但是rmid在释放时并不会立即可用，只有当该rmid的cache占用量低于了某个水位时，该rmid才会取消dirty标记表示可用。`rmid_dirty`检查当前rmid的cache占用量，如果超过`intel_cqm_threshold`时就说明为dirty。

```c
/*
 * As of now the RMIDs allocation is global.
 * However we keep track of which packages the RMIDs
 * are used to optimize the limbo list management.
 */
int alloc_rmid(void)
{
    struct rmid_entry *entry;
    // rmid的管理是一个全局操作，需要加锁
    lockdep_assert_held(&rdtgroup_mutex);
    // 如果当前的空闲rmid为空就会检查rmid_limbo_count，
    // rmid_limbo_count是一个计数器，对系统中没有使用的rmid
    // 但是cache占用量超过intel_cqm_threshold的rmid进行统计（潜在的可用RMID）
    if (list_empty(&rmid_free_lru))
        return rmid_limbo_count ? -EBUSY : -ENOSPC;

    // 如果有空闲的则返回空闲的
    entry = list_first_entry(&rmid_free_lru,
                 struct rmid_entry, list);
    list_del(&entry->list);

    return entry->rmid;
}

static bool rmid_dirty(struct rmid_entry *entry)
{
    u64 val = __rmid_read(entry->rmid, QOS_L3_OCCUP_EVENT_ID);

    return val >= intel_cqm_threshold;
}

```

##### free rmid

`free_rmid`释放一个rmid，但是不一定立刻加入到free-list中去。如果当前rdt支持llc-occupancy监控则检查后rmid当前的cache占用量，进行进一步的处理，如果rdt不支持llc-occupancy监控功能，当然就不存在对cache的占用量分析，立刻就返回到free-list中去。

```c
void free_rmid(u32 rmid)
{
    struct rmid_entry *entry;

    if (!rmid)
        return;
    // 加全局锁
    lockdep_assert_held(&rdtgroup_mutex);
    // 获取rmid对应的entry
    entry = __rmid_entry(rmid);
    // 如果启用了llc的占用量监控，则会对rmid的llc占用量进行分析，看能否加入自由链表
    // 如果没有开启直接返回队列
    if (is_llc_occupancy_enabled())
        add_rmid_to_limbo(entry);
    else
        list_add_tail(&entry->list, &rmid_free_lru);
}
```

`add_rmid_to_limbo`会对每一个domain进行检查，如果当前cpu是属于该domain的cpu，则对该rmid在该domain的llc的占用量进行分析，如果超过水位说明，rmid不能立即释放，如果该domain上没有busy的rmid还会额外开启一个定时任务，一段时间后再次检查rmid的llc占用量。如果在所有的domain中该rmid都没有占用超过`intel_cqm_threshold`的llc，此时才会将rmid加入自由链表。

```c
static void add_rmid_to_limbo(struct rmid_entry *entry)
{
    struct rdt_resource *r;
    struct rdt_domain *d;
    int cpu;
    u64 val;

    r = &rdt_resources_all[RDT_RESOURCE_L3];

    entry->busy = 0;
    cpu = get_cpu();
    list_for_each_entry(d, &r->domains, list) {
        if (cpumask_test_cpu(cpu, &d->cpu_mask)) {
            val = __rmid_read(entry->rmid, QOS_L3_OCCUP_EVENT_ID);
            if (val <= intel_cqm_threshold)
                continue;
        }

        /*
         * For the first limbo RMID in the domain,
         * setup up the limbo worker.
         */
        if (!has_busy_rmid(r, d))
            cqm_setup_limbo_handler(d, CQM_LIMBOCHECK_INTERVAL);
        set_bit(entry->rmid, d->rmid_busy_llc);
        entry->busy++;
    }
    put_cpu();

    if (entry->busy)
        rmid_limbo_count++;
    else
        list_add_tail(&entry->list, &rmid_free_lru);
}
```

##### 初始化

可以看到有一个参数`intel_cqm_threshold`会影响到rmid的回收，该参数会在初始化时进行设置。该值有cache的大小、资源的rmid总数以及`x86_cache_occ_scale`相关。

```c
int rdt_get_mon_l3_config(struct rdt_resource *r)
{
    int ret;

    r->mon_scale = boot_cpu_data.x86_cache_occ_scale;
    r->num_rmid = boot_cpu_data.x86_cache_max_rmid + 1;

    /*
     * A reasonable upper limit on the max threshold is the number
     * of lines tagged per RMID if all RMIDs have the same number of
     * lines tagged in the LLC.
     *
     * For a 35MB LLC and 56 RMIDs, this is ~1.8% of the LLC.
     */
    intel_cqm_threshold = boot_cpu_data.x86_cache_size * 1024 / r->num_rmid;

    /* h/w works in units of "boot_cpu_data.x86_cache_occ_scale" */
    intel_cqm_threshold /= r->mon_scale;

    ret = dom_data_init(r);
    if (ret)
        return ret;

    l3_mon_evt_init(r);

    r->mon_capable = true;
    r->mon_enabled = true;

    return 0;
}
```

### resctrl的文件操作与内核实现

相关的代码主要在`/arch/x86/kernel/cpu/intel_rdt_rdtgroup.c`中。
为了更好的理解resctrl的资源分配机制，更好的方式是通过resctrl文件系统的操作出发，从resctrl文件系统的操作对应到内核中的实现，其中最核心的操作包括

1. 创建CTRL-MON group和创建MON group
2. 指定CTRL-MON group和MON group的cpus
3. 修改CTRL-MON group的schemata
4. 在CTRL-MON group、MON group之间移动task。

#### rdt group的创建

在resctrl根目录下可以创建rdt group，可以看到rdt group创建的过程分为两部分：

1. 如果cpu支持资源分配，比如cat、mba等功能，并且是在resctrl根目录下创建目录则会调用`rdtgroup_mkdir_ctrl_mon`创建一个CTRL-MON group。
2. 如果支持监控功能，并且是在mon_groups目录下则会调用`rdtgroup_mkdir_mon`创建一个MON group。

```c
static int rdtgroup_mkdir(struct kernfs_node *parent_kn, const char *name,
              umode_t mode)
{
    /* Do not accept '\n' to avoid unparsable situation. */
    if (strchr(name, '\n'))
        return -EINVAL;

    /*
     * If the parent directory is the root directory and RDT
     * allocation is supported, add a control and monitoring
     * subdirectory
     */
    if (rdt_alloc_capable && parent_kn == rdtgroup_default.kn)
        return rdtgroup_mkdir_ctrl_mon(parent_kn, parent_kn, name, mode);

    /*
     * If RDT monitoring is supported and the parent directory is a valid
     * "mon_groups" directory, add a monitoring subdirectory.
     */
    if (rdt_mon_capable && is_mon_groups(parent_kn, name))
        return rdtgroup_mkdir_mon(parent_kn, parent_kn->parent, name, mode);

    return -EPERM;
}
```

在`rdtgroup_mkdir_ctrl_mon`和`rdtgroup_mkdir_mon`中都会分配一个closid，并且都会调用`mkdir_rdt_prepare`函数分配一个rmid。因此在resctrl视角下，每一个根目录下创建的目录和mon_groups目录下创建的目录都是一个控制组+监控组，但是mon_groups下的MON group并不能实现资源分配。

```c
rdtgroup_mkdir
    -->rdtgroup_mkdir_ctrl_mon
        -->mkdir_rdt_prepare
            -->alloc_rmid
        -->closid_alloc
    -->rdtgroup_mkdir_mon
        -->mkdir_rdt_prepare
            -->alloc_rmid
        -->closid_alloc
```

#### 修改cpus文件

指定cpus按照目录的类型不同分为两种操作，当group类型为CTRL-MON group时会调用cpus_ctrl_write，目录类型为MON group时会调用cpus_mon_write。通过对源码的分析，处理的过程需要进行权限检查和对cpu的默认closid和rmid的重新设置。

对于CTRL-MON group来说：

1. drop掉的cpu需要还给默认CTRL-MON group，因此默认CTRL-MON group无法进行drop cpu；
2. 对于新增的cpu需要更新这些cpu的默认closid和rmid；对于这些新增cpu的原CTRL-MON group需要移除这些cpu；
3. 对于CTRL-MON group下的MON group需要清空cpu_mask，MON group清空掉的cpu需要更新对应默认closid和rmid为parent CTRL-MON group。

对于MON group来说：

1. 首先会检查新增的cpus是否属于parent CTRL-MON group，MON group不允许添加parent CTRL-MON group中不存在的cpu；
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

#### 修改schedmata

`rdtgroup_schemata_write`会对输入进行解析，按照不同的资源分别通过IPI的方式进行更新，`rdt_ctrl_update`修改closid对应的cbm（cache bit mask）以及delay。

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

#### 在CTRL-MON group、MON group之间移动task

移动task会导致更新task绑定的closid和rmid更新为rdt group的closid和rmid，如果task加入的新group是CTRL-MON group，则task指定的closid和rmid都会更新，如果加入的新group是MON group，则只会更新rmid。

```c
rdtgroup_tasks_write
    -->rdtgroup_move_task(pid, rdtgrp, of)
        -->__rdtgroup_move_task(tsk, rdtgrp)

    
static int __rdtgroup_move_task(struct task_struct *tsk,
                struct rdtgroup *rdtgrp)
{
    // 检查task是否已经在group中
    if ((rdtgrp->type == RDTCTRL_GROUP && tsk->closid == rdtgrp->closid &&
         tsk->rmid == rdtgrp->mon.rmid) ||
        (rdtgrp->type == RDTMON_GROUP && tsk->rmid == rdtgrp->mon.rmid &&
         tsk->closid == rdtgrp->mon.parent->closid))
        return 0;
    // 如果移入一个ctrl group，更新task的closid和rmid
    // 如果移入一个mon group，需要检查task是否属于mon group的parent ctrl group
    // 否则就只更新task的rmid
    if (rdtgrp->type == RDTCTRL_GROUP) {
        tsk->closid = rdtgrp->closid;
        tsk->rmid = rdtgrp->mon.rmid;
    } else if (rdtgrp->type == RDTMON_GROUP) {
        if (rdtgrp->mon.parent->closid == tsk->closid) {
            tsk->rmid = rdtgrp->mon.rmid;
        } else {
            rdt_last_cmd_puts("Can't move task to different control group\n");
            return -EINVAL;
        }
    }

    update_task_closid_rmid(tsk);

    return 0;
}
```

### 监控事件的记录

内核代码主要在`/arch/x86/kernel/cpu/intel_rdt_monitor.c`
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

### task换入

比较清晰的是，task在换入时会将task所属的cosid和rmid写入到pqr_assoc寄存器中，指导内核进行cache等资源的使用以及对资源的使用量进行统计。

```c
struct intel_pqr_state {
    u32   cur_rmid;   // 当前的rmid
    u32   cur_closid;   // 当前的closid
    u32   default_rmid;  // 用户分配的cpu默认的rmid
    u32   default_closid;  // 用户分配的cpu默认的closid
};

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
