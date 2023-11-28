<!-- BUG: bpf_perf_event_ouput报错EOPNOTSUPP -->

## 问题现象

`bpf_perf_event_ouput`调用失败后可能出现的现象有：
1. 用户态存在一些预期中事件没有输出，但是`lost_cb`没有触发
2. `bpf_perf_event_ouput`返回`EOPNOTSUPP`

```c
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(int));
    __uint(value_size, sizeof(u32));
} map SEC(".maps");

bpf_perf_event_output(ctx, &map, 0, &ev, sizeof(ev));
```

## 问题原因和解决方法

`BPF_MAP_TYPE_PERF_EVENT_ARRAY`这种map类型是per-cpu的缓冲区需要指定output的缓冲区位置，错误代码`flags`中没有指定index为`BPF_F_CURRENT_CPU`，因此当`event->oncpu != cpu`不相同时就会报错。

```c

static __always_inline u64
__bpf_perf_event_output(struct pt_regs *regs, struct bpf_map *map,
			u64 flags, struct perf_sample_data *sd)
{
	struct bpf_array *array = container_of(map, struct bpf_array, map);
	unsigned int cpu = smp_processor_id();
	u64 index = flags & BPF_F_INDEX_MASK;
	struct bpf_event_entry *ee;
	struct perf_event *event;

	if (index == BPF_F_CURRENT_CPU)
		index = cpu;
	if (unlikely(index >= array->map.max_entries))
		return -E2BIG;

	ee = READ_ONCE(array->ptrs[index]);
	if (!ee)
		return -ENOENT;

	event = ee->event;
	if (unlikely(event->attr.type != PERF_TYPE_SOFTWARE ||
		     event->attr.config != PERF_COUNT_SW_BPF_OUTPUT))
		return -EINVAL;

	if (unlikely(event->oncpu != cpu))
		return -EOPNOTSUPP;

	perf_event_output(event, sd, regs);
	return 0;
}
```

正确的使用方式应该设置标志位参数`flags`为`BPF_F_CURRENT_CPU`.

```c
bpf_perf_event_output(ctx, &map, BPF_F_CURRENT_CPU, &ev, sizeof(ev));
```