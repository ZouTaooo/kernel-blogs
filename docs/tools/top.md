#! https://zhuanlan.zhihu.com/p/647047481
# top

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1682300206455-47bfb3ab-6225-4e75-83da-89aad8e876fe.png#clientId=ub8f3dfdc-2f01-4&from=paste&height=293&id=uc73eb6d1&originHeight=586&originWidth=1792&originalType=binary&ratio=2&rotation=0&showTitle=false&size=690463&status=done&style=none&taskId=u9ae7341b-b97d-4057-bc4b-f7be2437012&title=&width=896)

## 报告输出格式

- Summary Area
- Column Header
- Task Area

## Linux内存类型

1. 物理内存
2. swap file 如果被修改可以保存并且后续可以被获取
3. 虚拟内存 只受限于地址空间 主要作用如下：
   1. 虚拟 不受物理内存地址的限制
   2. 隔离 不同的进程间地址空间独立
   3. 共享 一次映射可以满足多个进程访问
   4. 可拓展 可以把虚拟地址赋予文件

所有的内存类型的都是以页为单位进行管理 OS处理物理内存和swap文件的方式是一致
每一个进程拥有的每一个内存页都是属于以下四种类型

- Anonymous和File-backed的区别在于是否基于文件 如果是可写内存会存在脏页写回
- Private和Shared的区别在于是否可以让多个进程共享

```python
                         Private | Shared
                     1           |          2
Anonymous  . stack               |
           . malloc()            |
           . brk()/sbrk()        | . POSIX shm*
           . mmap(PRIVATE, ANON) | . mmap(SHARED, ANON)
          -----------------------+----------------------
             . mmap(PRIVATE, fd) | . mmap(SHARED, fd)
File-backed  . pgms/shared libs  |
                     3           |          4

```

## 命令参数

```python
-b 批操作 可以将output重定向到其他程序或者文件 这个模式下top不接受输入 会按照 -n 指示的迭代次数进行迭代输出
-c 显示进程的命令及参数
-d 刷新间隔
-E k|m|g|t|p|e 改变summary area的单位 可选分别对应KB MB GB TB PB EB 
-H 线程模式 显示单独的线程 默认将一个进程的多个线程聚合显示
-i 显示从上次更新以后没有使用CPU的idle进程
-n number 迭代次数
-o filedname 选择列名进行排序 在列名前加+号从高到低排序 -号相反
-p pids 指示top的task pid
-s 安全模式？
-S 开启后累计进程和其dead的子进程的cpu时间
-u 指定uid 或者 username
-w 指定width 可以用于batch模式
```

## 报告参数

### summary area

```python
1. UPTIME and LOADAVG
系统的启动时间-用户数量-1、5、15分钟的平均负载

2. TASK and CPU state
第一行： task或者thread总数（和是否开启-H模式有关）- 以及各个状态的task或者thread数量（running；sleeping；stopped；zombie）

第二行：CPU时间的分布
   us, user    : time running un-niced user processes
   sy, system  : time running kernel processes
   ni, nice    : time running niced user processes
   id, idle    : time spent in the kernel idle handler
   wa, IO-wait : time waiting for I/O completion
   hi :   time spent servicing hardware interrupts
   si :   time spent servicing software interrupts
   st :   time stolen from this vm by the hypervisor

3. Memory Usage
第一行反映物理内存信息
total-free-used-buff/cache（用户缓存写入fs和从fs中读取出数据的内存）

第二行反应虚拟内存（swap file）信息
total-free-used-avail（估计可使用的物理内存）
```

### Column Header and Task Area

```bash
  PID      process id
  USER     username
  NI       nice值
  VIRT     虚拟内存使用量
  RES      进程正在使用的非交换的物理内存总量（VIRT的一部分）
  SHR      共享内存总量（RES的一部分）
  S        进程状态
  %CPU     CPU占比
  %MEM     内存占比
  TIME+    运行时间
  COMMAND  命令名
```
