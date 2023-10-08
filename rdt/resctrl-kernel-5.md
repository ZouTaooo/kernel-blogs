<!-- #! https://zhuanlan.zhihu.com/p/647042986 -->
<!-- # Resctrl内核实现（五）在group之间迁移task -->
## 前言   

task的写入会导致task绑定的CLOSID和RMID改变，本文对Resctrl中task的迁移过程进行了分析。

## 在CTRL-MON group、MON group之间移动task

对tasks的写操作会触发`rdtgroup_move_task`，调用`__rdtgroup_move_task`。
移动task会导致更新task绑定的closid和rmid更新为rdt group的closid和rmid。如果task加入的新group是CTRL-MON group，则task指定的closid和rmid都会更新，如果加入的新group是MON group，则只会更新rmid。

另外task在写入MON group的tasks文件时还会检查task是否属于parent-CTRL-MON group，否则会迁移失败。因此另一个CTRL-MON group的task在迁移到另一个CTRL-MON group下的MON group时只能先从前一个CTRL-MON group迁移到另一个CTRL-MON group中，再从CTRL-MON group迁移到MON group，不允许直接跨CTRL-MON group迁移。

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
