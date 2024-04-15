<!-- #! https://zhuanlan.zhihu.com/p/647048160 -->
# iotop

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2023/png/94756340/1681983785117-1b475a88-a383-45d3-a75d-749981b051d4.png#clientId=u82a2e39e-c9c1-4&from=paste&height=172&id=uabd27f90&originHeight=344&originWidth=1444&originalType=binary&ratio=2&rotation=0&showTitle=false&size=415876&status=done&style=none&taskId=u4b5243b8-b83b-4cb0-8409-d130c336bf3&title=&width=722)


## 命令参数

```bash
-o 		--only			只显示在做IO的进程
-b 		--batch			启用非交互模式 不断输出iotop信息 用于日志记录
-n NUM 		--iter=NUM		可以与-b一起使用 输出NUM次
-d SEC 		--delay=SEC		设置迭代输出的间隔
-p PIDs		--pid=PID		指定一组进程或者线程
-u USERs				指定用户
-P		--processes		只显示进程 默认显示所有线程
-a		--accumulated		累计IO而不是带宽 这个模式下显示从iotop开始时的累积IO数量
-k		--kilobytes		统一使用kB作为单位 默认会自动使用适合的单位B KB MB G
-t		--time			添加timestamp 默认会添加--batch
-q		--quiet			默认开启--batch
                        -q 	只在第一次迭代打印列名
                                    -qq 	不打印列名
                                    -qqq 	不输出IO summary
```

## 报告参数

```bash
Total DISK READ/WRITE	进程、内核线程与内核的设备子系统之间的IO带宽
Actual DISK READ/WRITE	内核的设备子系统与硬件外设之间的IO带宽
PRIO			线程的IO优先级 可以通过ionice命令设置
SWAPIN和IO		线程在swapin和等待IO的时间占比
```
