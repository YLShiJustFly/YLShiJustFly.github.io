随着OSD的更替和集群的扩缩容，PG在OSD的分布会逐渐变的不均衡，导致各OSD的实际容量使用率出现差异，集群整体使用率降低。ceph balancer模块就是通过调整权重或者upmap指定pg映射来让pg分布均匀的模块，分为upmap模式和crush-compat模式，本文基于Pacific版本，主要分析和使用upmap模式的运行流程。
# balancer模块运行流程
blancer模块执行流程概览：
![](../../image/mgr-balancer-1.jpg)
ceph 的 balancer 分为 upmap 和 crush-compat两种模式，只能采用一种设置，对比两种模式 ：
## 实现方式
crush-compat 会在 crush map 中生成单独的choose_args 列表，包含调整过后的权重集，依靠该列表调整数据的分布，执行 `ceph crush dump`可以查看， 例如：
```bash
    "choose_args": {
        "-1": [
            {
                "bucket_id": -1,
                "weight_set": [
                    [
                        0.1983489990234375,
                        0.1943206787109375,
                        0.1934051513671875
                    ]
                ]
            },
            {
                "bucket_id": -2,
                "weight_set": [
                    [
                        0.1983489990234375,
                        0.1943206787109375,
                        0.1934051513671875
                    ]
                ]
            },
            {
                "bucket_id": -3,
                "weight_set": [
                    [
                        0.1060638427734375,
                        0.09228515625
                    ]
                ]
            },
            {
                "bucket_id": -4,
                "weight_set": [
                    [
                        0.1060638427734375,
                        0.09228515625
                    ]
                ]
            },
            {
                "bucket_id": -5,
                "weight_set": [
                    [
                        0.096160888671875,
                        0.0981597900390625
                    ]
                ]
            },
            {
                "bucket_id": -6,
                "weight_set": [
                    [
                        0.096160888671875,
                        0.0981597900390625
                    ]
                ]
            },
            {
                "bucket_id": -7,
                "weight_set": [
                    [
                        0.0999908447265625,
                        0.093414306640625
                    ]
                ]
            },
            {
                "bucket_id": -8,
                "weight_set": [
                    [
                        0.0999908447265625,
                        0.093414306640625
                    ]
                ]
            }
        ]
    }
```
upmap 模式会在 osd map 中生成 pg 的映射关系，例如第一条表示，将 PG 3.1e 从 OSD5 迁移到 OSD2 上：
```bash
pg_upmap_items 3.1e [5,2]
pg_upmap_items 3.27 [4,0]
pg_upmap_items 3.28 [5,2]
pg_upmap_items 3.29 [5,2]
pg_upmap_items 3.33 [5,2]
pg_upmap_items 3.3e [5,2]
```
## 版本
1. crush-compat 模式兼容所有版本的客户端，客户端来请求OSDMap和CRUSH map，会使用choose_args结构（balancer调整后生成）中的权重。
2. upmap 模式不支持 L 版本以下的客户端。
## 影响范围
crush-compat 模式根据权重控制 OSD 分布，集群会根据该规则对 PG 做重映射，无法控制影响的 PG 范围。
upmap 根据用户设置的可容忍最大 PG 偏离数和每周期最多可以调整多少个 PG 来控制 PG 重映射影响的范围。
# upmap模式运行流程
打开 OSD 日志 `ceph tell mgr.* config set debug_osd 30/30`，对比 upmap 执行代码流程进行分析，蓝色为日志打印。
图中的流程表示：在双副本存储池下，把 pg 3.28 的 up_set 从 [3,5] 改为 [3,2]，即把 pg 3.28 从 osd.5 移到 osd.2。
![](../../image/mgr-balancer-2.jpg)
# 参数
## 时间参数
```bash
mgr/balancer/begin_time: 开始的时间，格式为HM，例如 0000                                       mgr/balancer/end_time：结束时间，格式为HM，例如 0100                                           mgr/balancer/begin_weekday：拜几开始，可取值1、2、3、4、5、6、7，例如 2
mgr/balancer/end_weekday：礼拜几结束，可取值1、2、3、4、5、6、7，例如 7
mgr/balancer/sleep_interval：balancer休眠多少秒后执行调整操作，例如 180
```
## upmap 关联的控制参数
```bash
mgr/balancer/active：是否启用balancer模块，true/false，例如 true
mgr/balancer/mode：balancer模式，分upmap和crush-compat，例如 upmap
mgr/balancer/upmap_max_deviation：允许偏离几个OSD，例如 5                                     mgr/balancer/upmap_max_optimizations：每次开始balancer最多调优多少轮退出，例如 10
```
## crush-compat 关联的控制参数
```bash
mgr/balancer/crush_compat_max_iterations：按照指定步长最多可调整多少次，例如 25                 mgr/balancer/crush_compat_metrics：参与score计算的指标，例如pgs,objects,bytes                 mgr/balancer/crush_compat_step：权重调整的步长，控制调整的权重精确度，例如0.500000               mgr/balancer/min_score：要调整到小于等于该score才表示调整完成，例如 0.020000                     mgr/balancer/mode：调整模式，例如crush-compat
```
# 执行balancer
设置`mgr/balancer/upmap_max_deviation`为2，等待balancer执行完成。
调整前：
```bash
[root@ceph-01 ~]# ceph osd df tree
ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA     OMAP     META      AVAIL    %USE   VAR   PGS  STATUS  TYPE NAME       
-1         0.58612         -  600 GiB   81 GiB   74 GiB  408 MiB   6.4 GiB  519 GiB  13.50  1.00    -          root default    
-3         0.19537         -  200 GiB   27 GiB   25 GiB  139 MiB   2.1 GiB  173 GiB  13.56  1.00    -              host ceph-01
 2    hdd  0.09769   1.00000  100 GiB   11 GiB  9.7 GiB   72 MiB  1008 MiB   89 GiB  10.76  0.80   23      up          osd.2   
 5    hdd  0.09769   1.00000  100 GiB   16 GiB   15 GiB   67 MiB   1.1 GiB   84 GiB  16.36  1.21   31      up          osd.5   
-5         0.19537         -  200 GiB   30 GiB   27 GiB   92 MiB   2.8 GiB  170 GiB  15.09  1.12    -              host ceph-02
 1    hdd  0.09769   1.00000  100 GiB   15 GiB   13 GiB   37 MiB   1.5 GiB   85 GiB  14.84  1.10   28      up          osd.1   
 4    hdd  0.09769   1.00000  100 GiB   15 GiB   14 GiB   55 MiB   1.3 GiB   85 GiB  15.35  1.14   29      up          osd.4   
-7         0.19537         -  200 GiB   24 GiB   22 GiB  177 MiB   1.6 GiB  176 GiB  11.86  0.88    -              host ceph-03
 0    hdd  0.09769   1.00000  100 GiB   11 GiB  9.8 GiB  107 MiB   1.1 GiB   89 GiB  10.99  0.81   25      up          osd.0   
 3    hdd  0.09769   1.00000  100 GiB   13 GiB   12 GiB   70 MiB   501 MiB   87 GiB  12.72  0.94   27      up          osd.3   
                       TOTAL  600 GiB   81 GiB   74 GiB  408 MiB   6.4 GiB  519 GiB  13.50                                     
MIN/MAX VAR: 0.80/1.21  STDDEV: 2.15
```
调整后：
```bash
[root@ceph-01 ~]# ceph osd df tree
ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA    OMAP     META     AVAIL    %USE   VAR   PGS  STATUS  TYPE NAME       
-1         0.58612         -  600 GiB   82 GiB  74 GiB  408 MiB  7.0 GiB  518 GiB  13.60  1.00    -          root default    
-3         0.19537         -  200 GiB   27 GiB  25 GiB  139 MiB  2.3 GiB  173 GiB  13.70  1.01    -              host ceph-01
 2    hdd  0.09769   1.00000  100 GiB   14 GiB  13 GiB   72 MiB  1.3 GiB   86 GiB  13.93  1.02   28      up          osd.2   
 5    hdd  0.09769   1.00000  100 GiB   13 GiB  12 GiB   67 MiB  1.1 GiB   87 GiB  13.47  0.99   26      up          osd.5   
-5         0.19537         -  200 GiB   28 GiB  25 GiB   92 MiB  2.8 GiB  172 GiB  14.17  1.04    -              host ceph-02
 1    hdd  0.09769   1.00000  100 GiB   14 GiB  13 GiB   37 MiB  1.5 GiB   86 GiB  14.23  1.05   27      up          osd.1   
 4    hdd  0.09769   1.00000  100 GiB   14 GiB  13 GiB   55 MiB  1.3 GiB   86 GiB  14.10  1.04   27      up          osd.4   
-7         0.19537         -  200 GiB   26 GiB  24 GiB  177 MiB  1.8 GiB  174 GiB  12.93  0.95    -              host ceph-03
 0    hdd  0.09769   1.00000  100 GiB   13 GiB  12 GiB  107 MiB  1.4 GiB   87 GiB  13.14  0.97   28      up          osd.0   
 3    hdd  0.09769   1.00000  100 GiB   13 GiB  12 GiB   70 MiB  501 MiB   87 GiB  12.72  0.94   27      up          osd.3   
                       TOTAL  600 GiB   82 GiB  74 GiB  408 MiB  7.0 GiB  518 GiB  13.60                                     
MIN/MAX VAR: 0.94/1.05  STDDEV: 0.54
```
OSD 上的 PG 数量偏差已经小于等于设置的 2，CRUSH 规则没有被改变，OSD 的容量也变得均衡了。
