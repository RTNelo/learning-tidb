# 对 TiDB 进行基准测试

## 环境

环境整体部署在阿里云的 ECS 上

### 机器配置

PD: (ecs.c5.large) 2vCPU, 4GB RAM, 40GB ESSD 云盘 (PL0, 单盘 IOPS 上限 10k), 内网带宽 1Gbps.

TiKV: (ecs.c5.2xlarge) 8vCPU, 16GB RAM, 500GB ESSD 云盘 (PL1, 单盘 IOPS 上限 50k), 内网带宽 1Gbps.

TiDB: (ecs.c5.2xlarge) 8vCPU, 16GB RAM, 40GB ESSD 云盘 (PL0, 单盘 IOPS 上限 10k), 内网带宽 1Gbps.

监控节点: (ecs.c5.xlarge) 4vCPU, 8GB RAM, 40GB ESSD 云盘 (PL0, 单盘 IOPS 上限 10k), 内网带宽 1Gbps.

以上节点均安装了 Ubuntu 18.04 LTS.

### 拓扑结构

集群中包含 PD x 3 + TiKV x 3 + TiDB x 2 + 监控节点 x 1.
其中 PD, TiKV, TiDB 分别部署一个实例到一台对应的机器上. `monitoring_server` `alertmanager_server` `grafana_server` 均部署到唯一的监控节点上.

以上节点均连接到一个 `192.168.0.0/16` 的私有网络中.

具体的 TiUP 配置如下:

```
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"

pd_servers:
  - host: 192.168.0.162
  - host: 192.168.0.163
  - host: 192.168.0.164

tidb_servers:
  - host: 192.168.0.156
  - host: 192.168.0.158

tikv_servers:
  - host: 192.168.0.159
  - host: 192.168.0.160
  - host: 192.168.0.161

monitoring_servers:
  - host: 192.168.0.165

grafana_servers:
  - host: 192.168.0.165

alertmanager_servers:
  - host: 192.168.0.165
```

## 执行测试

### sysbench

#### 准备

使用如下配置: 

```
mysql-host=192.168.0.156
mysql-port=4000
mysql-user=root
mysql-password=
mysql-db=sbtest
time=600
threads=32
report-interval=10
db-driver=mysql
```

使用如下命令, 准备测试数据:

```
$ sysbench --config-file=sysbench.conf oltp_point_select --tables=32 --table-size=10000 prepare
```

#### 执行

执行如下测试命令

```
$ sysbench --config-file=sysbench.conf oltp_point_select --threads=16 --tables=32 --table-size=10000 run

$ sysbench --config-file=sysbench.conf oltp_update_index --threads=16 --tables=32 --table-size=10000 run

$ sysbench --config-file=sysbench.conf oltp_read_only --threads=16 --tables=32 --table-size=10000 run
```

#### 结果

sysbench 最终结果输出如下:

```
SQL statistics:
    queries performed:
        read:                            20471574
        write:                           0
        other:                           0
        total:                           20471574
    transactions:                        20471574 (34118.84 per sec.)
    queries:                             20471574 (34118.84 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0063s
    total number of events:              20471574

Latency (ms):
         min:                                    0.31
         avg:                                    0.47
         max:                                   32.57
         95th percentile:                        0.60
         sum:                              9590815.79

Threads fairness:
    events (avg/stddev):           1279473.3750/7409.85
    execution time (avg/stddev):   599.4260/0.01
```

```
SQL statistics:                                                                                                                                                                                                                               
    queries performed:                                                                                                                                                                                                                        
        read:                            0                                                                                                                                                                                                    
        write:                           1431296                                                                                                                                                                                              
        other:                           0                                                                                                                                                                                                    
        total:                           1431296                                                                                                                                                                                              
    transactions:                        1431296 (2385.45 per sec.)                                                                                                                                                                           
    queries:                             1431296 (2385.45 per sec.)                                                                                                                                                                           
    ignored errors:                      0      (0.00 per sec.)                                                                                                                                                                               
    reconnects:                          0      (0.00 per sec.)                                                                                                                                                                               
                                                                                                                                                                                                                                              
General statistics:                                                                                                                                                                                                                           
    total time:                          600.0085s                                                                                                                                                                                            
    total number of events:              1431296                                                                                                                                                                                              
                                                                                                                                                                                                                                              
Latency (ms):                                                                                                                                                                                                                                 
         min:                                    3.21                                                                                                                                                                                         
         avg:                                    6.71                                                                                                                                                                                         
         max:                                  221.95                                                                                                                                                                                         
         95th percentile:                        8.28                                                                                                                                                                                         
         sum:                              9599168.21                                                                                                                                                                                         
                                                                                                                                                                                                                                              
Threads fairness:                                                                                                                                                                                                                             
    events (avg/stddev):           89456.0000/210.33                                                                                                                                                                                          
    execution time (avg/stddev):   599.9480/0.00
```

```
SQL statistics:
    queries performed:
        read:                            8177064
        write:                           0
        other:                           1168152
        total:                           9345216
    transactions:                        584076 (973.43 per sec.)
    queries:                             9345216 (15574.84 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.0184s
    total number of events:              584076

Latency (ms):
         min:                                    9.35
         avg:                                   16.43
         max:                                  109.23
         95th percentile:                       22.69
         sum:                              9599092.68

Threads fairness:
    events (avg/stddev):           36504.7500/80.48
    execution time (avg/stddev):   599.9433/0.01
```

TiDB 的 Query Summary 的 QPS:

![TiDB 的 Query Summary 的 QPS](https://pic3.zhimg.com/80/v2-c2551f8425e8eea1f2860899ea6e3b6b_1440w.png)

TiDB 的 Query Summary 的 Duration:

![TiDB 的 Query Summary 的 Duration](https://pic3.zhimg.com/80/v2-abcf30f4eea984ba50fffac9c65c6cee_1440w.png)

TiKV Details 的 Cluster 的 CPU:

![TiKV Details 的 Server 的 CPU](https://pic4.zhimg.com/80/v2-34d1ccf2f1266b115a1606c04d2451ae_1440w.png)

TiKV Details 的 Cluster 的 QPS:

![TiKV Details 的 Server 的 QPS](https://pic1.zhimg.com/80/v2-d27a484bd54e23d0876465ebdd65c6dd_1440w.png)

TiKV Details 的 gRPC 的 message count:

![TiKV Details 的 gRPC 的 message count](https://pic2.zhimg.com/80/v2-23c1d4ddee7fbf1865f81d4f2dc7f64b_1440w.png)

TiKV Details 的 gRPC 的 message duration:

![TiKV Details 的 gRPC 的 message duration](https://pic2.zhimg.com/80/v2-23c1d4ddee7fbf1865f81d4f2dc7f64b_1440w.png)

### ycsb

#### 执行

执行如下命令

```
$ bin/go-ycsb load mysql -P workloads/workloada -p recordcount=1500000 -p mysql.host=192.168.0.158 -p mysql.port=4000 --threads 16
```

#### 结果

go-ycsb 输出如下

```
***************** properties *****************
"insertproportion"="0"
"workload"="core"
"dotransactions"="false"
"readproportion"="0.5"
"updateproportion"="0.5"
"scanproportion"="0"
"threadcount"="16"
"recordcount"="1500000"
"mysql.host"="192.168.0.158"
"mysql.port"="4000"
"readallfields"="true"
"operationcount"="1000"
"requestdistribution"="uniform"
**********************************************
INSERT - Takes(s): 10.0, Count: 97313, OPS: 9730.7, Avg(us): 1616, Min(us): 534, Max(us): 18373, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 15000
INSERT - Takes(s): 20.0, Count: 196437, OPS: 9821.8, Avg(us): 1601, Min(us): 518, Max(us): 21199, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 17000
INSERT - Takes(s): 30.0, Count: 294939, OPS: 9831.2, Avg(us): 1599, Min(us): 518, Max(us): 21199, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 17000
INSERT - Takes(s): 40.0, Count: 392408, OPS: 9810.2, Avg(us): 1603, Min(us): 518, Max(us): 26100, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 17000
INSERT - Takes(s): 50.0, Count: 489851, OPS: 9797.0, Avg(us): 1605, Min(us): 507, Max(us): 26100, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 16000
INSERT - Takes(s): 60.0, Count: 588369, OPS: 9806.1, Avg(us): 1603, Min(us): 507, Max(us): 26100, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 16000
INSERT - Takes(s): 70.0, Count: 686673, OPS: 9809.6, Avg(us): 1603, Min(us): 507, Max(us): 26100, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 16000
INSERT - Takes(s): 80.0, Count: 785113, OPS: 9813.9, Avg(us): 1602, Min(us): 507, Max(us): 26100, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 16000
INSERT - Takes(s): 90.0, Count: 885614, OPS: 9840.1, Avg(us): 1598, Min(us): 507, Max(us): 26100, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 17000
INSERT - Takes(s): 100.0, Count: 984435, OPS: 9844.2, Avg(us): 1597, Min(us): 507, Max(us): 26100, 99th(us): 7000, 99.9th(us): 12000, 99.99th(us): 17000
INSERT - Takes(s): 110.0, Count: 1019501, OPS: 9268.2, Avg(us): 1698, Min(us): 507, Max(us): 145959, 99th(us): 8000, 99.9th(us): 13000, 99.99th(us): 20000
INSERT - Takes(s): 120.0, Count: 1041911, OPS: 8682.6, Avg(us): 1814, Min(us): 507, Max(us): 169696, 99th(us): 9000, 99.9th(us): 14000, 99.99th(us): 26000
INSERT - Takes(s): 130.0, Count: 1064680, OPS: 8189.8, Avg(us): 1925, Min(us): 507, Max(us): 169696, 99th(us): 9000, 99.9th(us): 15000, 99.99th(us): 29000
INSERT - Takes(s): 140.0, Count: 1087626, OPS: 7768.7, Avg(us): 2031, Min(us): 507, Max(us): 169696, 99th(us): 10000, 99.9th(us): 15000, 99.99th(us): 29000
INSERT - Takes(s): 150.0, Count: 1110180, OPS: 7401.2, Avg(us): 2133, Min(us): 507, Max(us): 169696, 99th(us): 10000, 99.9th(us): 16000, 99.99th(us): 28000
INSERT - Takes(s): 160.0, Count: 1133114, OPS: 7082.0, Avg(us): 2231, Min(us): 507, Max(us): 169696, 99th(us): 10000, 99.9th(us): 16000, 99.99th(us): 28000
INSERT - Takes(s): 170.0, Count: 1154687, OPS: 6792.3, Avg(us): 2327, Min(us): 507, Max(us): 169696, 99th(us): 11000, 99.9th(us): 17000, 99.99th(us): 60000
INSERT - Takes(s): 180.0, Count: 1177642, OPS: 6542.4, Avg(us): 2417, Min(us): 507, Max(us): 169696, 99th(us): 11000, 99.9th(us): 17000, 99.99th(us): 60000
INSERT - Takes(s): 190.0, Count: 1200839, OPS: 6320.2, Avg(us): 2503, Min(us): 507, Max(us): 169696, 99th(us): 11000, 99.9th(us): 17000, 99.99th(us): 57000
INSERT - Takes(s): 200.0, Count: 1223924, OPS: 6119.6, Avg(us): 2586, Min(us): 507, Max(us): 169696, 99th(us): 11000, 99.9th(us): 17000, 99.99th(us): 56000
INSERT - Takes(s): 210.0, Count: 1246881, OPS: 5937.5, Avg(us): 2666, Min(us): 507, Max(us): 169696, 99th(us): 11000, 99.9th(us): 17000, 99.99th(us): 56000
INSERT - Takes(s): 220.0, Count: 1268770, OPS: 5767.1, Avg(us): 2746, Min(us): 507, Max(us): 259435, 99th(us): 11000, 99.9th(us): 18000, 99.99th(us): 67000
INSERT - Takes(s): 230.0, Count: 1291964, OPS: 5617.2, Avg(us): 2820, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 18000, 99.99th(us): 66000
INSERT - Takes(s): 240.0, Count: 1314607, OPS: 5477.5, Avg(us): 2892, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 18000, 99.99th(us): 65000
INSERT - Takes(s): 250.0, Count: 1337265, OPS: 5349.1, Avg(us): 2962, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 18000, 99.99th(us): 64000
INSERT - Takes(s): 260.0, Count: 1359692, OPS: 5229.6, Avg(us): 3031, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 19000, 99.99th(us): 77000
INSERT - Takes(s): 270.0, Count: 1378162, OPS: 5104.3, Avg(us): 3106, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 21000, 99.99th(us): 98000
INSERT - Takes(s): 280.0, Count: 1400944, OPS: 5003.4, Avg(us): 3169, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 21000, 99.99th(us): 98000
INSERT - Takes(s): 290.0, Count: 1424110, OPS: 4910.7, Avg(us): 3229, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 21000, 99.99th(us): 97000
INSERT - Takes(s): 300.0, Count: 1447171, OPS: 4823.9, Avg(us): 3288, Min(us): 507, Max(us): 259435, 99th(us): 12000, 99.9th(us): 21000, 99.99th(us): 96000
INSERT - Takes(s): 310.0, Count: 1469888, OPS: 4741.6, Avg(us): 3345, Min(us): 507, Max(us): 259435, 99th(us): 13000, 99.9th(us): 21000, 99.99th(us): 96000
INSERT - Takes(s): 320.0, Count: 1491192, OPS: 4660.0, Avg(us): 3404, Min(us): 507, Max(us): 259435, 99th(us): 13000, 99.9th(us): 22000, 99.99th(us): 98000
INSERT - Takes(s): 330.0, Count: 1500000, OPS: 4545.5, Avg(us): 3424, Min(us): 507, Max(us): 259435, 99th(us): 13000, 99.9th(us): 22000, 99.99th(us): 98000
Run finished, takes 5m30.609936688s
INSERT - Takes(s): 330.6, Count: 1500000, OPS: 4537.1, Avg(us): 3424, Min(us): 507, Max(us): 259435, 99th(us): 13000, 99.9th(us): 22000, 99.99th(us): 98000
```

TiDB 的 Query Summary 的 QPS:

![TiDB 的 Query Summary 的 QPS](https://pic4.zhimg.com/80/v2-d57000aef0ab63b6aa9cf81ca328c10c_1440w.png)

TiDB 的 Query Summary 的 Duration:

![TiDB 的 Query Summary 的 Duration](https://pic4.zhimg.com/80/v2-7dd2e3da8ba91bc0f36c1e1d2feb43de_1440w.png)

TiKV Details 的 Cluster 的 CPU:

![TiKV Details 的 Server 的 CPU](https://pic2.zhimg.com/80/v2-2ed07495c205f6a30337e4923a4e07eb_1440w.png)

TiKV Details 的 Cluster 的 QPS:

![TiKV Details 的 Server 的 QPS](https://pic1.zhimg.com/80/v2-0a2cef07be6c3e15941bbe6f0a346919_1440w.png)

TiKV Details 的 gRPC 的 message count:

![TiKV Details 的 gRPC 的 message count](https://pic1.zhimg.com/80/v2-efbb1b2dc59c1614fa19f65b4ca25473_1440w.png)

TiKV Details 的 gRPC 的 message duration:

![TiKV Details 的 gRPC 的 message duration](https://pic4.zhimg.com/80/v2-799148ed2f6666df5780e6909012dee1_1440w.png)

### tpcc

#### 执行

执行以下命令:

```
$ bin/go-tpc tpcc -H 192.168.0.158 -P 4000 -D tpcc --warehouses 50 prepare
```

#### 结果

TiDB 的 Query Summary 的 QPS:

![TiDB 的 Query Summary 的 QPS](https://pic3.zhimg.com/80/v2-cbc52bdcc1ea9c15d8984289ffdaa2bc_1440w.png)

TiDB 的 Query Summary 的 Duration:

![TiDB 的 Query Summary 的 Duration](https://pic3.zhimg.com/80/v2-ca8aa27450ec4df965bbcad47bfb9e30_1440w.png)

TiKV Details 的 Cluster 的 CPU:

![TiKV Details 的 Server 的 CPU](https://pic1.zhimg.com/80/v2-d6755b8561c751ac538d90d8eb75b174_1440w.png)

TiKV Details 的 Cluster 的 QPS:

![TiKV Details 的 Server 的 QPS](https://pic3.zhimg.com/80/v2-ff882cdd866d99ac773f971a369bd01f_1440w.png)

TiKV Details 的 gRPC 的 message count:

![TiKV Details 的 gRPC 的 message count](https://pic1.zhimg.com/80/v2-7d7a65a30b64abf32323a919f4fe1fe0_1440w.png)

TiKV Details 的 gRPC 的 message duration:

![TiKV Details 的 gRPC 的 message duration](https://pic3.zhimg.com/80/v2-3274c7647ed525c347d149ebea31d684_1440w.png)


## 可能的性能瓶颈

在查询过程中, TiDB 服务器的 CPU 利用率较高, 推测性能瓶颈位于 TiDB 上.
