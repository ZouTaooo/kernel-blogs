<!-- BUG: bpf_trace_printk调用在trace文件中看不到输出 -->

## 问题现象

在写一个eBPF程序时发现调用`bpf_trace_printk`后，编译运行都顺利进行但是在trace_pipe和trace文件中中看不到任何相关输出。

```c
char fmt[32] = "debug: send pid=(%d) duration=(%d)\n";
bpf_trace_printk(fmt, sizeof(fmt), pid, ev.duration_ns);
```

## 问题原因和解决方法

`bpf_trace_printk`的使用方式不对。正确的使用方式应该去掉`fmt`的数组长度限制，这样`sizeof`计算得到的长度才是包含`\0`在内的完整字符串长度。`bpf_trace_printk`对该值的限制比较严格不会手动检测`\0`位置。

```c
char fmt[] = "debug: send pid=(%d) duration=(%d)\n";
bpf_trace_printk(fmt, sizeof(fmt), pid, ev.duration_ns);
```