# imc-latency event summary

## Intel Core Processor & Intel Atom Processor & Intel Xeon Phi Processor

均不支持

## Intel Xeon Processor

### Haswell EP & Ivy Bridge EP & Broadwell EP

| event       | evsel | umask | desc                           |
| ----------- | ----- | ----- | ------------------------------ |
| RPQ_INSERTS | 0x10  | 0x00  | Read Pending Queue Allocations |

### Sky Lake-X(SKX) & Cascade Lake-X & Sandy Bridge EP（Jake Town）

| event         | evsel | umask | desc                            |
| ------------- | ----- | ----- | ------------------------------- |
| RPQ_INSERTS   | 0x10  | 0x00  | Read Pending Queue Allocations  |
| RPQ_OCCUPANCY | 0x80  | 0x00  | Read Pending Queue Occupancy    |
| WPQ_INSERTS   | 0x20  | 0x00  | Write Pending Queue Allocations |
| WPQ_OCCUPANC  | 0x81  | 0x00  | Write Pending Queue Occupancy   |

### Ice Lake(ICX) & Sapphire Rapids(SPR) & Snow Ridge

| event              | evsel | umask     | desc                            |
| ------------------ | ----- | --------- | ------------------------------- |
| RPQ_INSERTS.PCH0   | 0x10  | bxxxxxxx1 | Read Pending Queue Allocations  |
| RPQ_INSERTS.PCH1   | 0x10  | bxxxxxx1x | Read Pending Queue Allocations  |
| RPQ_OCCUPANCY_PCH0 | 0x80  | 0x00      | Read Pending Queue Occupancy    |
| RPQ_OCCUPANCY_PCH1 | 0x81  | 0x00      | Read Pending Queue Occupancy    |
| WPQ_INSERTS.PCH0   | 0x20  | bxxxxxxx1 | Write Pending Queue Allocations |
| WPQ_INSERTS.PCH1   | 0x20  | bxxxxxx1x | Write Pending Queue Allocations |
| WPQ_OCCUPANCY_PCH0 | 0x82  | 0x00      | Write Pending Queue Occupancy   |
| WPQ_OCCUPANCY_PCH1 | 0x83  | 0x00      | Write Pending Queue Occupancy   |

### Wesetmere EX & Nehalem EX

| event | evsel | umask | desc |
| ----- | ----- | ----- | ---- |
| None  | None  | None  | None |

## 结论

只有`Xeon`系列的`Sky Lake(SKX)`、`Cascade Lake`、`Sandy Bridge`、`Ice Lake(ICX)`、`Sapphire Rapids(SPR)`、`Snow Ridge`等`CPU model`支持`uncore imc`的内存延迟指标计算。

## reference

[intel uncore event](https://perfmon-events.intel.com/#)
