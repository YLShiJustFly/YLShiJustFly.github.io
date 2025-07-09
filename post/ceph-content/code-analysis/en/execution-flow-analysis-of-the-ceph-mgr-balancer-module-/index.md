As OSD are replaced and the cluster scales in and out, the distribution of PGs across OSDs becomes increasingly unbalanced. This leads to discrepancies in actual usage rates of individual OSDs, reducing the overall utilization rate of cluster. The ceph balancer module addresses this by adjusting weights or specifying PG mappings via upmap to redistribute PGs evently. 

> This article analyzes the execution flow when using balancer upmap mode, based on the ceph Pacific version.

# balancer module execution flow

Overview diagram of the balancer module execution flow:

![](../../image/mgr-balancer-1-en.jpg)



The balancer has two modes: upmap and crush-compat.

Only one can be active at a time. Below is a comparison of two modes:

## Implementation method
`crush-compat`: Generates a separate `choose_args`list in the crush map, which contains adjusted weight sets used to influence PGs distribution. You can view it using `ceph crush dump`. Example:

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

`upmap`: Creates explicit PG-to-OSD mappings in the osdmap. For example, this line:

`pg_upmap_items 3.1e [5,2]` means PG 3.1e is remapped from osd.5 to osd.2.

You can view all mappings use `ceph osd dump`:

```bash
pg_upmap_items 3.1e [5,2]
pg_upmap_items 3.27 [4,0]
pg_upmap_items 3.28 [5,2]
pg_upmap_items 3.29 [5,2]
pg_upmap_items 3.33 [5,2]
pg_upmap_items 3.3e [5,2]
```

## Version compatibility
1. `crush-compat`: Compatible with all ceph client versions. Clients will use the ajusted weights in `choose_args`from the CRUSH map.
2. `upmap`: Not supported by clients older then version Luminous. 

## Scope of impact
`crush-compat`: Controls PG distribution by weight, but cannot limit which PGs are remapped. 

`upmap`: Allow configuration of the maximum number of PG deviation and the number of PGs that can be remmaped in each round of adjustment, offering finer control.

# upmap mode execution flow

Enable detailed OSD logs with `ceph tell mgr.* config set debug_osd 30/30`, the content within the blue box is log print information.

The diagram shows the execution flow of changing the `up_set`of PG 3.28 from [3,5] to [3,2] in a two replica pool, e.g., moving PG 3.28 from osd.5 to osd.2.

![](../../image/mgr-balancer-2-en.jpg)

# Parameters
## Time-related parameters
```bash
mgr/balancer/begin_time: start time (HHMM), e.g., 0000                                      mgr/balancer/end_time：end time (HHMM), e.g., 0100                                          mgr/balancer/begin_weekday：start weekday，can take value 1、2、3、4、5、6、7, e.g., 2
mgr/balancer/end_weekday：end weekday，can take value 1、2、3、4、5、6、7，e.g., 7
mgr/balancer/sleep_interval：time in seconds the balancer sleeps before running again，e.g., 180
```

## upmap related parameters
```bash
mgr/balancer/active：whether to enable the balancer module，true/false，e.g., true
mgr/balancer/mode：balancer mode (upmap or crush-compat)，e.g., upmap
mgr/balancer/upmap_max_deviation：Allowed max deviation in pg between osd，e.g., 5         mgr/balancer/upmap_max_optimizations：Maximum number of adjustments, e.g., 10
```

## crush-compat related parameters
```bash
mgr/balancer/crush_compat_max_iterations：Maximum number of iterations for adjustments，e.g., 25
mgr/balancer/crush_compat_metrics：metrics for score calculation，e.g., pgs,objects,bytes   mgr/balancer/crush_compat_step：step size for weight adjustment, e.g., 0.500000           mgr/balancer/min_score：target score to complete balancing，e.g., 0.020000                 mgr/balancer/mode：balancer mode，e.g., crush-compat
```

# Testing
run`ceph config set mgr mgr/balancer/upmap_max_deviation 2` command，waiting for balancer to complete......

Before balancing：

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

After balancing：

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

The deviation in the number of PGs on the OSDs is now less than or equal to the set value of 2, the CRUSH rule have not been changed, and the utilization rate of the OSDs has become balanced.
