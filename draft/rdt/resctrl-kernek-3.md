<!-- #! https://zhuanlan.zhihu.com/p/647040743 -->
<!-- # Resctrl内核实现（三）GROUP的创建 -->
## 前言

在（一）我们已经清楚了Resctrl中RMID和CLOSID的切换规则是怎样的。
在（二）中对内核中的CLOSID和RMID的分配、释放过程进行了详细解读。
在后续的章节将会对Resctrl文件系统中重要的文件操作触发的内核行为进行解读。本章将对Resctrl中的建组操作进行分析。

## rdt group的创建

在Resctrl中有两类建组操作，一类是在根目录下创建CTRL-MON group，另一类则是在CTRL-MON group下的mon_groups目录下创建MON-group。
在内核中Resctrl文件系统中的创建目录的处理函数是`rdtgroup_mkdir`。主要逻辑如下：

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

相比于建组操作，对RDT有一定了解的肯定更关心建组操作是如何分配CLOSID和RMID。首先说结论：创建CTRL-MON group会分配一个CLOSID和RMID，创建一个MON group只会分配RMID，MON group会复用其parent CTRL-MON group的closid。

`rdtgroup_mkdir_ctrl_mon`和`rdtgroup_mkdir_mon`的函数调用链如下。

```c
rdtgroup_mkdir
    -->rdtgroup_mkdir_ctrl_mon
        -->mkdir_rdt_prepare
            -->alloc_rmid
        -->closid_alloc
    -->rdtgroup_mkdir_mon
        -->mkdir_rdt_prepare
            -->alloc_rmid
```
