# 内存延迟指标

## read meomroy latency metric

### RPQ_INSERTS

- Title-Read Pending Queue Allocations
- Max Inc/Cyc 1
- Definition-统计分配进入读等待队列的数量。Read Pending Queue用于调度读请求到内存控制器，并追踪这些请求。读请求在从HA发送到IMC之前需要进入RPQ，并且需要在缓冲区分配一定空间。这些空间在CAS命令被发送到内存后释放。

### RPQ_OCCUPANCY

- Title-Read Pending Queue Occupancy
- Max Inc/Cyc 22 // 和WPQ-32不一致？
- Definition-累积每个周期的RPQ占用量。这个累计值可以用于和队列非空的周期数一起计算平均占用量，也可以和分配次数一起计算平均延迟。

### 读延迟计算

avg_clk=RPQ_OCCUPANCY/RPQ_INSERTS 每个读请求的平均周期数
clk_per_second=clock/1s DRAM每秒的周期数
read_latency = avg_clk/clk_per_second = 每个读请求的延迟时间

## write memory latency metric

### WPQ_INSERTS

- Title-Write Pending Queue Allocations
- Max Inc/Cyc 1
- Definition-统计分配进入写等待队列的数量。Write Pending Queue用于调度写请求到内存控制器，并追踪这些请求。写请求在从HA发送到IMC之前需要进入RPQ，并且需要在缓冲区分配一定空间。这些空间在CAS命令被发送到内存后释放。

### WPQ_OCCUPANCY

- Title-Write Pending Queue Occupancy
- Max Inc/Cyc 32
- Definition-累积每个周期的WPQ占用量。这个累计值可以用于和队列非空的周期数一起计算平均占用量，也可以和分配次数一起计算平均延迟。

### 写延迟计算

avg_clk=WPQ_OCCUPANCY/WPQ_INSERTS 每个写请求的平均周期数
clk_per_second=clock/1s DRAM每秒的周期数
write_latency = avg_clk/clk_per_second = 每个写请求的延迟时间

## ref

1. xeon-e5-2600-uncore-guide
