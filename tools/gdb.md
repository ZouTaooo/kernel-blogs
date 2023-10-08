<!-- GDB的常用方法 -->
## 前言
GDB，the GNU Project Debugger，一种命令行调试工具。这里我将遇到的一些重要用法记录下来。

## 使用GDB前的准备

编译选项需要加上`-g -O0`，用于产生调试信息，并且禁止优化（可能编译结果与源代码信息不匹配）。

## 断点和观察点
断点，也就是break point，当程序运行到断点时会暂停，可以查看当前的程序状态。观察点，用于监视某个变量或者是某个内存地址的内容是否被修改，当出现变化时和断点一样会暂停运行。

打断点的方式有两种，通过行号和符号名。

```bash
b {file-name}:line-num      # file-name是可选项
b {file-name}:function-name 

sample:
    b main  # 断点在main函数运行前
    b 8     # 断点在当前文件的第8行
```

打一个观察点有三种方式，常用的就是监视某个变量和地址监视，监视变量时需要保证变量处于上下文中，需要在程序运行以后再打观察点。
```bash
sample:
    wathc val                   # 监视变量val
    wathc *(int *)0x12345678    # 监视地址0x12345678处的四字节内存内容
    wathc `a*b + c/d`           # 监控一个表达式，操作符需要满足程序的原生语言
```

`info b`查看当前所有的断点+观察点。包括断点编号（Num）、类型（Type）、是否启用（Enb）等等。

```bash
(gdb) info b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000000400549 in main at main.c:8
        breakpoint already hit 1 time
3       hw watchpoint  keep n                      c
        breakpoint already hit 1 time
```

`disable {Num}`，禁用某个断点使用`disable`+断点号。
`enable {Num}`，启用某个断点使用`enable`+断点号。

## 携带参数执行

可以进入gdb以后，在run命令后携带。比如一个可执行文件`foo`运行：`foo --arg1 val1 --arg2 val2`.
```bash
sample:
    gdb foo
    # 进入gdb后
    run --arg1 val1 --arg2 val2
```

## 单步执行和按行执行

单步执行的命令是`step`,缩写`s`，当要执行的下一行存在函数嵌套时，可以使用单步执行进入到函数中，如果是一条普通语句则和按行执行相同。

按行执行的命令是`next`，缩写`n`，粒度相比于单步执行更大，一次执行一行。

## 连续执行

第一种，如果相同重复执行某个命令，不需要重复输入，只需要回车键就可以再次执行，适合连续单步执行或者按行执行。

第二种，如果想要一直运行到下一个断点处，可以使用`continue`，缩写`c`。

第三种，如果某个loop循环次数太多想要跳过一部分，希望可以循环`n`次，此时只需要在循环内打一个断点然后`continue n`.

第四种，如果想要运行到某一行停下来，但是又不想打断点，可以使用`until`。需要注意如果中间出现断点，还是会停下来。

## 函数的返回

如果想从当前函数中跳出返回到上一层有两种方式，`finish`会运行到函数上一层停止, 可以查看到函数的返回值。
```bash
(gdb) finish
Run till exit from #0  add (a=1, b=2) at main.c:4
0x000000000040056d in main () at main.c:10
10              c = add(a, b);
Value returned is $2 = 3
```

如果`finish`没有显示返回值还可以通过`print $eax`来查看（可能只在X86上生效，CPU的返回值寄存器可能不相同）。

还有一种`return`, `return`可以直接指定返回值的内容，不需要继续执行。
```bash
(gdb) return 4
Make add return now? (y or n) y
#0  0x000000000040056d in main () at main.c:10
10              c = add(a, b);
(gdb) p c
$3 = 0
(gdb) n
11              printf("c=%d\n", c);
(gdb) p c
$4 = 4
```

## 查看函数栈

`backtrace`缩写`bt`，打印当前函数栈。

```c
(gdb) bt
#0  add (a=1, b=2) at main.c:4
#1  0x000000000040056d in main () at main.c:10
```

如果想要查看上一层的栈的局部变量，可以通过`up 1`返回上一层。`down 1`回到下一层栈。需要注意`up`和`down`不会影响程序状态，只是切换了当前的栈信息。

## 查看某个变量的值和类型

`print`缩写`p`，可以查看某个变量的值。
`ptype`可以查看某个变量的类型。

```bash
(gdb) p c
$13 = 3
(gdb) ptype c
type = int
```
## coredump

针对某些难复现的bug，如果需要程序运行一段时间才会出现，此时可以通过设置coredump，程序挂掉后会产生coredump文件保存现场进行debug。coredump的启用需要设置ulimit、coredump文件名格式以及存储路径。

coredump现场可以通过`gdb exe-file coredump-file`的方式进行恢复。调试方式与普通程序是一致的。

## 多线程、多进程调试

平常用的不多，后续再补充。