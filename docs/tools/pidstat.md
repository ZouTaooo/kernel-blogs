<!-- #! https://zhuanlan.zhihu.com/p/647049204 -->
# pidstat

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1682042280708-60371494-dd7c-4bf0-93f3-3ce4577abc4c.png#clientId=u223988e9-2c10-4&from=paste&height=332&id=ubc4cdab2&originHeight=664&originWidth=1898&originalType=binary&ratio=2&rotation=0&showTitle=false&size=855419&status=done&style=none&taskId=ub795e203-46d4-460a-9acc-5cfc42b591d&title=&width=949)

## 命令参数

```bash
-C comm   只显示包含comm的task comm可以是正则表达式
-d   指定IO利用率 包括 UID USER PID kB_rd/s kB_wr/s kB_ccwr/s iodelay Command
-e program args  执行 program args 并用pidstat监视 program停止时pidstat停止
-G process-name  只显示包含process-name的进程 process-name可以是正则表达式 如果同时使用了-t 则属于process的线程也会显示
-H    显示时间戳
-h    显示所有的活动在一行中 在报告结尾不添加平均统计 方便其他程序解析
--human   提高单位人类可读性
-l    显示prcess command和参数
-p { pid [,...] | SELF | ALL } 指定pid SELF表示pidstat自己 ALL表示全部
-R   指定报告的实时优先级信息 包括 UID USER PID prio policy Command
-r   报告页错误和内存利用率 包括 UID USER PID minflt/s majflt/s VSZ RSS %MEM Command
   当报告任务和其子任务的全局统计信息时 还可能会显示 UID USER PID minflt-nr majflt-nr Command
-s    报告栈利用率 包括 UID USER PID StkSize StkRef Command
-T {TASK|CHILD|ALL} 报告全局数据 而不是某个时刻的数据 TASK报告每一个task的全局信息 CHILD表示报告选择的task和其子task的全局信息   
-t    报告线程的归属信息 包括 TGID TID
-U username   显示用户名 而不是UID 如果指定了username只显示该用户的task信息
-u    显示CPU利用率 包括 UID USER PID %usr %system %guest %wait %CPU CPU Command
   当显示全局的统计信息时 包括 UID USER PID usr-ms system-ms guest-ms Command
-v    显示一些内核表 包括UID USER PID threads fd-nr Command
-w    显示task切换行为 包括UID USER PID cswch/s nvcswch/s Command
```

## 报告参数

### cpu

#### 瞬时数据

```bash
[weizhen.zt@localhost btf]$ pidstat -u
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时14分56秒   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
12时14分56秒     0         1    0.00    0.00    0.00    0.00    0.01     1  systemd
12时14分56秒     0         2    0.00    0.00    0.00    0.00    0.00     3  kthreadd
12时14分56秒     0        11    0.00    0.00    0.00    0.00    0.00     0  ksoftirqd/0
12时14分56秒     0        12    0.00    0.01    0.00    0.00    0.01     0  rcu_sched

UID  用户ID
PID  task id
%usr 用户空间cpu占比
%system 内核态cpu占比
%guest  虚拟机内的task的cpu占比
%wait 等待运行的cpu占比
```

#### 全局数据

```bash
[weizhen.zt@localhost btf]$ pidstat -T ALL -u
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时57分21秒   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
12时57分21秒     0         1    0.00    0.00    0.00    0.00    0.01     2  systemd
12时57分21秒     0         2    0.00    0.00    0.00    0.00    0.00     0  kthreadd
12时57分21秒     0        11    0.00    0.00    0.00    0.00    0.00     0  ksoftirqd/0
12时57分21秒     0        12    0.00    0.01    0.00    0.00    0.01     3  rcu_sched
...
12时57分21秒   UID       PID    usr-ms system-ms  guest-ms  Command
12时57分21秒     0         1   1131690    284480         0  systemd
12时57分21秒     0         2         0       240         0  kthreadd
12时57分21秒     0        11        20       650         0  ksoftirqd/0
12时57分21秒     0        12         0     15960         0  rcu_sched
12时57分21秒     0        13        20      1640         0  migration/0
12时57分21秒     0        16        10      1730         0  migration/1

usr-ms  用户态时间
system-ms 内核态时间
guest-ms task和其children在虚拟机中运行的时间
```

### IO

```bash
[weizhen.zt@localhost btf]$ pidstat -d
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时19分17秒   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
12时19分17秒     0         1     -1.00     -1.00     -1.00      10  systemd
12时19分17秒     0        39     -1.00     -1.00     -1.00       4  khugepaged
12时19分17秒     0       166     -1.00     -1.00     -1.00      28  kswapd0
12时19分17秒     0       651     -1.00     -1.00     -1.00      68  xfsaild/dm-0
12时19分17秒     0       747     -1.00     -1.00     -1.00      33  systemd-journal
12时19分17秒     0       783     -1.00     -1.00     -1.00       2  systemd-udevd

UID  用户ID
PID   task id
kB_rd/s  disk读取速率  
kB_wr/s  disk写入速率
kB_ccwr/s  被task取消的写入disk的大小每秒 会发生在task 清空一些脏页cache 在这种情况下 已经被考虑的其他task的IO可能不会发生
iodelay  块IO的延迟 单位是时钟滴答 包括等待同步块IO的完成和等待换入块IO的完成

```

### 实时优先级

```bash
[weizhen.zt@localhost btf]$ pidstat -R
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时29分59秒   UID       PID prio policy  Command
12时29分59秒     0        13   99   FIFO  migration/0
12时29分59秒     0        16   99   FIFO  migration/1
12时29分59秒     0        21   99   FIFO  migration/2
12时29分59秒     0        26   99   FIFO  migration/3

prio 优先级 
policy  调度策略
```

### 内存

#### 瞬时数据

```bash
[weizhen.zt@localhost btf]$ pidstat -r
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时33分35秒   UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
12时33分35秒     0         1      0.73      0.00  170044    6804   0.18  systemd
12时33分35秒     0       747      0.02      0.00   45768    9036   0.24  systemd-journal
12时33分35秒     0       783      0.02      0.00   48640    3332   0.09  systemd-udevd
12时33分35秒     0       886      0.01      0.00  262836    3140   0.08  sssd
12时33分35秒     0       887      0.01      0.00  450012    3840   0.10  upowerd
12时33分35秒     0       888      0.00      0.00   11384    2272   0.06  smartd
12时33分35秒    81       890      0.01      0.00   19808    4160   0.11  dbus-daemon

minflt/s   minor页错误（还在内存 需要重新建立地址映射）数每秒
majflt/s   major页错误（需要从disk读到内存）数每秒
VSZ        虚拟内存使用size
RSS     没有被换出的内存size
%MEM  任务当前使用的有效物理内存占比
```

#### 全局数据

```bash
[weizhen.zt@localhost btf]$ pidstat -T ALL -r
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时55分00秒   UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
12时55分00秒     0         1      0.73      0.00  170044    6804   0.18  systemd
12时55分00秒     0       747      0.02      0.00   45768    9036   0.24  systemd-journal
12时55分00秒     0       783      0.02      0.00   48640    3332   0.09  systemd-udevd
...
12时55分00秒   UID       PID minflt-nr majflt-nr  Command
12时55分00秒     0         1  36415481    171202  systemd
12时55分00秒     0       747      3759       512  systemd-journal
12时55分00秒     0       783     47794       366  systemd-udevd
12时55分00秒     0       886      1234       197  sssd
12时55分00秒     0       887      1403        66  upowerd

minflt-nr  minor页错误数
majflt-nr major页错误数

```

### 栈

```bash
[weizhen.zt@localhost btf]$ pidstat -s
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时38分22秒   UID       PID StkSize  StkRef  Command
12时38分22秒  1000      2005     132      12  systemd
12时38分22秒  1000      2020     132      16  pulseaudio
12时38分22秒  1000      2021     132       4  pipewire
12时38分22秒  1000      2083     132       8  dbus-daemon
12时38分22秒  1000      2168     132       4  gdm-x-session
12时38分22秒  1000      2182     132       4  gnome-session-b
12时38分22秒  1000      2242     132       4  at-spi-bus-laun

StkSize  task预留的栈大小
StkRef  task使用的栈大小
```

### 内核表

```bash
[weizhen.zt@localhost btf]$ pidstat -v
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时41分20秒   UID       PID threads   fd-nr  Command
12时41分20秒  1000      2005       1      24  systemd
12时41分20秒  1000      2020       2      30  pulseaudio
12时41分20秒  1000      2021       2      29  pipewire
12时41分20秒  1000      2083       1      65  dbus-daemon
12时41分20秒  1000      2168       3       8  gdm-x-session

threads  线程数
fd-nr  fd数
```

### 进程切换

```bash
[weizhen.zt@localhost btf]$ pidstat -w
Linux 5.10.134-13.an8.aarch64 (localhost.localdomain)  2023年04月15日  _aarch64_ (4 CPU)

12时41分55秒   UID       PID   cswch/s nvcswch/s  Command
12时41分55秒     0         1      0.07      0.02  systemd
12时41分55秒     0         2      0.01      0.00  kthreadd
12时41分55秒     0         3      0.00      0.00  rcu_gp
12时41分55秒     0         4      0.00      0.00  rcu_par_gp
12时41分55秒     0         6      0.00      0.00  kworker/0:0H-events_highpri
12时41分55秒     0         8      0.00      0.00  mm_percpu_wq
12时41分55秒     0         9      0.00      0.00  rcu_tasks_rude_
12时41分55秒     0        10      0.00      0.00  rcu_tasks_trace
12时41分55秒     0        11      0.08      0.00  ksoftirqd/0
12时41分55秒     0        12      3.86      0.00  rcu_sched

cswch/s  自愿的上下文切换：出现在block在请求资源上
nvcswch/s 非自愿的上下文切换：被抢占
```
