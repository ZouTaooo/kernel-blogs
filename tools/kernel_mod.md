<!-- 编写运行一个基础内核模块的基本模版 -->
## 前言

内核模块是运行在内核态代码的一种形式，可以动态的装入和卸载。本文对编写和运行一个内核模块的最基础组件和模版进行了记录。

## 环境准备

编译需要依赖源码，编译如果出现build目录不存在，需要安装对应的kernel-devel包。如果包管理器中不存在对应版本的包请自行下载安装。

```bash
yum install kernel-devel-$(uname -r) kernel-headers-$(uname -r)  # 版本需要与当前的 kernel 版本对应
cd /lib/modules/`uname -r`/  # 查看 build 链接是否正常，若不正常，需要安装对应版本的 kernel-devel
```

为了防止模块导致hang机无法恢复，需要启用一些panic配置.

```bash
echo 1 > softlockup_panic
echo 1 > hardlockup_panic
echo 1 > panic_on_oops 
```

## 内核模块模版

```c

#define pr_fmt(fmt) "[%s]-[%s]: " fmt, "xxx", __func__

#include <linux/kernel.h>
#include <linux/module.h>

#define KSYM_NAME_LEN 16

static char symbol[KSYM_NAME_LEN] = "${mod_name}";
module_param_string(symbol, symbol, KSYM_NAME_LEN, 0644);

static int __init mod_init(void) {
    pr_info("Module init success.\n");
    return 0;
}

// static inline void proc_remove(struct proc_dir_entry *de)

static void __exit mod_exit(void) {
    pr_info("Module exit success.\n");
}

module_init(mod_init) module_exit(mod_exit) MODULE_LICENSE("GPL");
```

## Makefile模版

```bash
obj-m += ${mod_name}.o

all:
	make -C /usr/lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /usr/lib/modules/$(shell uname -r)/build M=$(PWD) clean

install:
	sudo insmod ${mod_name}.ko

uninstall:
	sudo rmmod ${mod_name}.ko

list:
	sudo lsmod
```

## 模块安装/卸载/查看

```bash
insmod ${mod_name}.ko
rmmod ${mod_name}
lsmod | grep ${mod_name}
```