<!-- #! https://zhuanlan.zhihu.com/p/647040311 -->
<!-- # Resctrl内核实现（二）CLOSID和RMID管理 -->
## 前言

RDT的监控数据累计和资源分配策略的关键就是CLOSID和RMID的分配策略。

## CLOSID和RMID管理

### CLOSID管理

RDT中的资源通过控制组进行分配，控制组对于各类资源进行划分或者限制。每一个控制组用一个CLOSID进行标识，由于CLOSID的数量有限所以在内核中通过一个位图进行表示和管理。

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

### RMID管理

Resctrl利用RMID定位一个监控组，进行事件的计数时会按照RMID统计到对应的监控组中，RMID在内核中通过一个链表进行管理，每个RMID使用`rmid_entry`结构体表示，`busy`标志位表示该RMID正在使用中。

```c
struct rmid_entry {
    u32    rmid;
    int    busy;
    struct list_head  list;
};
```

但是由于RMID的数量有限，就会涉及到RMID的分配与释放过程。

#### alloc rmid

`alloc_rmid`分配一个RMID，会从链表中找出一个空闲RMID。但是RMID在释放时并不会立即可用，只有当该RMID的llc occupancy低于了某个water-mark时，该RMID才会取消dirty标记变得可重用。`rmid_dirty`检查当前rmid的cache occupancy，如果超过`intel_cqm_threshold`时就说明RMID目前处于dirty状态。

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

#### free rmid

`free_rmid`释放一个RMID，但是该RMID不一定立刻加入到free-list中去。如果当前rdt支持llc occupancy监控则需要检查该RMID当前的llc occupancy进行进一步的处理；如果RDT不支持llc occupancy监控功能，当然就不存在对llc occupany的分析，此时立刻返回RMID到free-list中。

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

对llc occupancy进行进一步检查的操作由`add_rmid_to_limbo`完成。`add_rmid_to_limbo`会对每一个domain进行检查，如果当前cpu是属于该domain的cpu，则对该RMID在该domain的llc occupancy进行分析，如果超过水位说明RMID不能立即释放，同时检查该domain上是否有busy的RMID时，如果没有busy的RMID还会额外开启一个定时任务（需要再次检查，如果已有worker不需要再开启），一段时间后再次检查RMID的llc occupancy。如果在所有的domain中该RMID都没有占用超过`intel_cqm_threshold`的llc，此时才会将RMID加入自由链表。

**NOTE**：domain就是一个类似于socket的概念。

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

#### intel_cqm_threshold的初始化

可以看到有一个参数`intel_cqm_threshold`会影响到rmid的回收，该参数会在初始化时进行设置。该值与cache的大小、资源的rmid总数以及`x86_cache_occ_scale`相关。

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

## 总结

本章内核的RMID和CLOSID的分配、释放过程进行了详细的解读。核心点就是：

1. RMID和CLOSID是有限的，有可能分配失败
2. RMID可能不会立即释放，和llc occupany以及水位线有关
