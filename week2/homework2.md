# Homework 2

## 基本信息

- Week 2
- 日期：2020 年 8 月 23 日
- 姓名：何田丰
- 环境：Ubuntu 18.04 (Win 10 Hyper-V 虚拟机)

## 题目描述

使用 sysbench、go-ycsb 和 go-tpc 分别对 TiDB 进行测试并且产出测试报告。测试报告需要包括以下内容

- 部署环境的机器配置（CPU、内存、磁盘规格型号），拓扑结构（TiDB、TiKV 各部署于哪些节点）
- 调整过后的 TiDB 和 TiKV 配置
- 测试输出结果
- 关键指标的监控截图
  - TiDB Query Summary 中的 qps 与 duration
  - TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
  - TiKV Details 面板中 grpc 的 qps 以及 duration

输出：写出你对该配置与拓扑环境和 workload 下 TiDB 集群负载的分析，提出你认为的 TiDB 的性能的瓶颈所在（能提出大致在哪个模块即可）

## 部署环境

机器配置：

- CPU：Intel Core i7-6700 @ 3.40GHz， 4 核 8 线程
- 内存：DDR4 2133 16G
- 磁盘：TOSHIBA DT01ACA100，7200 转，150G(虚拟机)

拓扑环境：

- TiDB，TiKV，PD 全部部署于同一台虚拟机上
- 1 x TiDB，1 x PD，3 x TiKV

## TiDB 和 TiKV 配置

未作调整，默认配置

## 测试结果

\*_说明: 因为在个人电脑的虚拟机中测试，稍大一点的测试集 prepare 就卡很久，所以测试的数据集都很小。_

#### sysbench

测试命令

```
//准备
mysql> create database sbtest;
mysql> set global tidb_disable_txn_auto_retry=off;
mysql> set global tidb_txn_mode="optimistic";
sysbench olap_update_non_index --config-file=config --threads=4 --tables=4 --table_size=10000 prepare;

//测试
sysbench --config-file=config oltp_point_select --tables=4 --table-size=10000 run
sysbench --config-file=config oltp_update_index --tables=4 --table-size=10000 run
sysbench --config-file=config oltp_read_only --tables=4 --table-size=10000 run
```

oltp_point_select 测试结果

```
SQL statistics:
    queries performed:
        read:                            53311
        write:                           0
        other:                           0
        total:                           53311
    transactions:                        53311  (5329.63 per sec.)
    queries:                             53311  (5329.63 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0017s
    total number of events:              53311

Latency (ms):
         min:                                    0.12
         avg:                                    0.19
         max:                                   20.93
         95th percentile:                        0.30
         sum:                                 9968.27

Threads fairness:
    events (avg/stddev):           53311.0000/0.00
    execution time (avg/stddev):   9.9683/0.00
```

oltp_update_index 测试结果

```
SQL statistics:
    queries performed:
        read:                            0
        write:                           17
        other:                           0
        total:                           17
    transactions:                        17     (1.61 per sec.)
    queries:                             17     (1.61 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.5253s
    total number of events:              17

Latency (ms):
         min:                                  316.78
         avg:                                  619.10
         max:                                  949.49
         95th percentile:                      831.46
         sum:                                10524.78

Threads fairness:
    events (avg/stddev):           17.0000/0.00
    execution time (avg/stddev):   10.5248/0.00
```

oltp_read_only

```
SQL statistics:
    queries performed:
        read:                            21854
        write:                           0
        other:                           3122
        total:                           24976
    transactions:                        1561   (156.03 per sec.)
    queries:                             24976  (2496.44 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0033s
    total number of events:              1561

Latency (ms):
         min:                                    3.80
         avg:                                    6.40
         max:                                  138.06
         95th percentile:                       12.30
         sum:                                 9997.61

Threads fairness:
    events (avg/stddev):           1561.0000/0.00
    execution time (avg/stddev):   9.9976/0.00
```

#### go-ycsb

测试命令

```
//准备
mysql> create database test;
./bin/go-ycsb load mysql -P workloads/workloada -p recordcount=2000 -p mysql.host=127.0.0.1 -p mysql.port=4000 --threads 4
./bin/go-ycsb load mysql -P workloads/workloadb -p recordcount=2000 -p mysql.host=127.0.0.1 -p mysql.port=4000 --threads 4

//测试
./bin/go-ycsb run mysql -P workloads/workloada -p recordcount=2000 -p mysql.host=127.0.0.1 -p mysql.port=4000 --threads 4
./bin/go-ycsb run mysql -P workloads/workloadb -p recordcount=2000 -p mysql.host=127.0.0.1 -p mysql.port=4000 --threads 4
```

workloada 测试结果

```
Run finished, takes 2m22.313311397s
READ   - Takes(s): 142.3, Count: 471, OPS: 3.3, Avg(us): 5102, Min(us): 465, Max(us): 1058721, 99th(us): 8000, 99.9th(us): 1059000, 99.99th(us): 1059000
UPDATE - Takes(s): 141.5, Count: 529, OPS: 3.7, Avg(us): 1021027, Min(us): 467260, Max(us): 3036542, 99th(us): 2204000, 99.9th(us): 3037000, 99.99th(us): 3037000
```

workloadb 测试结果

```
Run finished, takes 14.648495227s
READ   - Takes(s): 14.6, Count: 938, OPS: 64.0, Avg(us): 1073, Min(us): 411, Max(us): 9129, 99th(us): 5000, 99.9th(us): 10000, 99.99th(us): 10000
UPDATE - Takes(s): 14.1, Count: 62, OPS: 4.4, Avg(us): 840327, Min(us): 411571, Max(us): 1337666, 99th(us): 1338000, 99.9th(us): 1338000, 99.99th(us): 1338000
```

#### go-tpc

测试命令

```
//准备
./bin/go-tpc tpcc -H 127.0.0.1 -P 4000 -D tpcc --warehouses 1 prepare
./bin/go-tpc tpch -H 127.0.0.1 -P 4000 -D tpch --sf 1 --analyze
//测试
./bin/go-tpc tpcc -H 127.0.0.1 -P 4000 -D tpcc --warehouses 1 run --time=1m --threads=4
./bin/go-tpc tpch -H 127.0.0.1 -P 4000 -D tpch --sf 1
```

TPC-C 测试结果

```
Finished
[Summary] DELIVERY - Takes(s): 32.5, Count: 2, TPM: 3.7, Sum(ms): 5489, Avg(ms): 2744, 90th(ms): 4000, 99th(ms): 4000, 99.9th(ms): 4000
[Summary] NEW_ORDER - Takes(s): 55.4, Count: 9, TPM: 9.7, Sum(ms): 24988, Avg(ms): 2776, 90th(ms): 8000, 99th(ms): 8000, 99.9th(ms): 8000
[Summary] ORDER_STATUS - Takes(s): 40.4, Count: 2, TPM: 3.0, Sum(ms): 498, Avg(ms): 249, 90th(ms): 512, 99th(ms): 512, 99.9th(ms): 512
[Summary] PAYMENT - Takes(s): 59.1, Count: 18, TPM: 18.3, Sum(ms): 30040, Avg(ms): 1668, 90th(ms): 4000, 99th(ms): 4000, 99.9th(ms): 4000
[Summary] PAYMENT_ERR - Takes(s): 59.1, Count: 3, TPM: 3.0, Sum(ms): 7183, Avg(ms): 2394, 90th(ms): 4000, 99th(ms): 4000, 99.9th(ms): 4000
[Summary] STOCK_LEVEL - Takes(s): 38.4, Count: 1, TPM: 1.6, Sum(ms): 685, Avg(ms): 685, 90th(ms): 1000, 99th(ms): 1000, 99.9th(ms): 1000
tpmC: 9.7
```

TPC-H 测试结果
数据准备卡在 generating orders/lineitem tables 一步，无法运行测试。单机性能伤不起啊 ToT。

## 关键指标监控

## 性能瓶颈分析

单击部署 3 个 Tikv ，性能瓶颈应该在 CPU。
