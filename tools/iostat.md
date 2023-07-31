#! https://zhuanlan.zhihu.com/p/647033826
# Linux工具-iostat

iostat命令可以查看IO设备的IO信息

## iostat报告预览

```python
$iostat
Linux 4.19.91-007.ali4000.alios7.x86_64 (VM20210331-84)         04/20/2023      _x86_64_        (4 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.77    0.01    1.79    0.01    0.10   96.32

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
vda              12.47         1.14       126.37   73868138 8190705216
vdb               0.00         0.00         0.00       2216          0
```

## iostat命令参数

```bash
-c 显示cpu报告
-d 显示设备报告
-g group-name <devs> 将多个dev聚集成group 除了会展示单个dev的信息外 还会将数据汇总显示为group
-H 和-g搭配使用 只显示汇总数据 不显示单个dev信息
-h 提高单位可读
-k 使用kb
-m 使用mb
-p <devs..> 指定devs
-x 显示拓展数据
```
## iostat报告参数

```bash
avg-cpu和top相同
tps		对IO设备每秒的请求次数
kB_read/s	每秒读的kB数
kB_wrtn/s	每秒写的kB数
kB_read		设备读字节总数
kB_wrtn		设备写字节总数
Device  	设备名          
r/s     	读请求（合并后）每秒完成数	
w/s     	写请求（合并后）每秒完成数
rkB/s    	同kB_read/s
wkB/s 		通kB_wrtn/s
rrqm/s   	每秒进入设备队列读请求数
wrqm/s 		每秒进入设备队列写请求数
%rrqm 		发送到设备前合并的读请求比例
%wrqm		发送到设备前合并的写请求比例
r_await		读请求发送到设备和设备处理时间的平均值
w_await		写请求发送到设备和设备处理时间的平均值
aqu-sz		设备的平均队列长度
rareq-sz 	读请求的平均大小
wareq-sz	写请求的平均大小
svctm 		平均服务时间 不可靠 将被弃用
%util		带宽利用率 接近100%说明设备饱和 对于可以并行处理的设备 该数值不能反映设备饱和状态
```

## iostat examples

```bash
iostat -d 2 // 每两秒输出一次报告
iostat -d 2 6 // 每两秒输出一次报告 一共输出六次
iostat -x sda sdb 2 6 // 为sda 和 sdb 每两秒输出一次拓展报告会 输出6次报告
iostat -p sda // 展示sda和它所有partition的报告
```
