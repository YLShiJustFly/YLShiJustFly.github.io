## 问题现象
- **环境**：Ceph集群节点异常重启后
- **异常指标**：`ceph_osd_op_w_latency`的Prometheus rate值
- **具体表现**：
  - 重启前：数值正常递增（峰值约1M）
  - 重启后：从0开始记录，3分钟后突增至42亿（接近2³²）
  - ![1cbdc1281f3f34a96a1cf40b55d69e44.jpg|600](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/1cbdc1281f3f34a96a1cf40b55d69e44.jpg)
## 排查过程
### 第一阶段：初始假设
| 假设方向               | 验证方法               | 结论                     |
|-----------------------|-----------------------|--------------------------|
| Prometheus计算问题    | 检查rate()函数逻辑    | 确认已处理计数器归零情况 |
| Ceph统计初始化问题    | 审查OSD.cc初始化代码  | 确认atomic变量正确初始化 |
### 第二阶段：深度分析
1. **关键发现**：
   - 跳变前存在3分钟归零期
   - 42亿跳变值导致坐标轴显示异常
2. **机制分析**：
   - 使用系统实时时钟计算时间差
   - 无负值保护机制
### 第三阶段：验证
1. **环境证据**：
   - NTP日志显示4ms时间回退
   - ![image.png|600](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20250608172327.png)
2. **复现方法**：
   - 通过fio施加IO负载
   - 手动回拨系统时间1ms
   - 成功复现跳变现象
   - ![](https://popofp.vipfp.ps.netease.com/file/684162b6d5dbab9ee1d1d45251JL0trO01)
## 根因结论
1. **直接原因**：
   - 系统时间回拨导致负时间差
   - 无符号整数存储转换
2. **深层原因**：
   - 未全面采用单调时钟
   - 相关PR #39273 <https://github.com/ceph/ceph/pull/39273> 没有修改全部的realtime
## 解决方案
1. **长期改进**：
   - 推动monotonic时钟全面替换
   - 或者增加时间差值校验逻辑
2. 在官方 slack 提了一个 issue <https://tracker.ceph.com/issues/71580>，目前官方已跟进修改 <https://github.com/ceph/ceph/pull/64003>。
