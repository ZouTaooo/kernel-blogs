#! https://zhuanlan.zhihu.com/p/647034304
# coredump-文件生成配置

核心转储文件，可以在程序dump时将当时的程序状态记录下，配合gdb工具进行分析。适合在长期运行的程序挂掉时进行分析。

## 开启coredump

### 设置coredump文件的limit

```shell
# 临时设置
ulimit -c unlimited

# 永久设置
vim /etc/security/limits.conf
# 去掉 soft core 0 一行前面的注释 并将0改为 unlimited
```

### 设置coredump文件目录

```shell
cat /proc/sys/kernel/core_pattern
echo /var/log/%e.core.%p > /proc/sys/kernel/core_pattern

# 参数列表
%p - insert pid into filename 添加pid
%u - insert current uid into filename 添加当前uid
%g - insert current gid into filename 添加当前gid
%s - insert signal that caused the coredump into the filename 添加导致产生core的信号
%t - insert UNIX time that the coredump occurred into filename 添加core文件生成时的unix时间
%h - insert hostname where the coredump happened into filename 添加主机名
%e - insert coredumping executable name into filename 添加命令名
```

## 编译选项

编译器取消编译器优化并且加上-g，保留符号信息，在debug时才能拿到正确的地址和行号。

```bash
gcc -g -O0 xxx
```
