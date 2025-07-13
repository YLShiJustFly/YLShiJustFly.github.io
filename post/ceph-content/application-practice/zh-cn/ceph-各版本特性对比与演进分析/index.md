# Ceph 各版本特性对比与演进分析

本文档详细分析了Ceph从Nautilus (v14)到最新Squid (v19)版本的主要特性演进，为选择合适版本和制定升级策略提供参考。(在LLM的帮助下整理)

## 版本概览

| 版本 | 代号 | 发布时间 | 生命周期状态 | 
|------|------|----------|--------------|
| v14.2.x | Nautilus | 2019年 | EOL |  |
| v15.2.x | Octopus | 2020年 | EOL |  |
| v16.2.x | Pacific | 2021年 | EOL |  |
| v17.2.x | Quincy | 2022年 | EOL |  |
| v18.2.x | Reef | 2023年 | 稳定维护版 | 
| v19.2.x | Squid | 2024年 | 当前稳定版 | 

---

## 版本特性对比总结

### 成熟度驱动的版本选择指南

| 特性 | Nautilus | Octopus | Pacific | Quincy | Reef | Squid |
|------|----------|---------|---------|---------|------|-------|
| **部署方式** | ceph-deploy | **cephadm引入** | cephadm | cephadm成熟 | cephadm | cephadm |
| **存储引擎** | BlueStore | BlueStore | BlueStore | BlueStore | **FileStore移除** | BlueStore优化 |
| **配置管理** | **集中化引入** | 集中化 | 集中化 | 集中化 | 集中化 | 集中化 |
| **网络协议** | **msgr2引入** | msgr2稳定 | msgr2 | msgr2 | msgr2 | msgr2 |
| **PG管理** | **autoscale引入** | autoscale | autoscale | autoscale | autoscale | autoscale |
| **调度器** | 传统 | 改进 | **mclock引入** | **mclock默认** | mclock | mclock优化 |
| **CephFS多FS** | **首次支持** | 功能增强 | **镜像完善** | 管理优化 | 管理优化 | Dashboard集成 |
| **多站点** | 基础 | **RBD镜像** | **CephFS镜像** | 完善 | 增强 | 增强 |
| **Dashboard** | 基础 | 改进 | 改进 | 改进 | **重构** | **重构** |
| **容器化** | 无 | **cephadm预览** | **cephadm成熟** | cephadm完整 | cephadm | cephadm |

#### 特性成熟度标记说明
- **加粗文字**: 该版本中特性的重要节点（首次引入/达到稳定/重大改进）
- 普通文字: 特性在该版本中保持稳定或有小幅改进
- 斜体文字: 特性在该版本中被弃用或准备移除

### 特性驱动的版本选择指南


| 所需特性 | 最低版本 | 稳定推荐版本 | 备注 |
|----------|----------|--------------|------|
| 中心化配置管理 | Nautilus | Octopus+ | 基础功能可用，建议升级获得稳定性 |
| PG autoscaling | Nautilus | Pacific+ | 生产环境建议手动控制 |
| msgr2安全协议 | Nautilus | Octopus+ | 新部署建议直接使用 |
| cephadm容器化管理 | Octopus | Pacific+ | 技术预览→生产可用 |
| CephFS多文件系统 | Nautilus | Pacific+ | 基础支持→生产可用 |
| CephFS镜像/灾备 | Octopus | Pacific+ | 功能引入→生产稳定 |
| mclock QoS调度 | Pacific | Quincy+ | 引入→默认启用 |
| 完全容器化部署 | Pacific | Quincy+ | 基础支持→完整生态 |
| 高级Dashboard | Reef | Squid+ | 重构→完善 |
| FileStore替换 | 任意版本 | Reef+ | Reef后FileStore不再支持 |


## 特性首次引入与成熟度分析

### 重要特性生命周期时间线

本节详细说明关键特性的首次引入版本、稳定版本以及推荐的生产环境采用时机，帮助用户做出明智的版本选择。

#### 配置管理演进
- **中心化配置管理**
  - **首次引入**: Nautilus (v14.x) - 基础功能可用
  - **稳定推荐**: Octopus (v15.x) - 修复了关键bug，生产环境可用
  - **技术价值**: 统一管理配置，支持在线修改，提供审计功能

#### 存储池管理革新
- **PG autoscaling/merging**
  - **首次引入**: Nautilus (v14.x) - 历史性突破，支持PG数量减少
  - **稳定推荐**: Pacific (v16.x) - 算法优化，减少误操作风险
  - **⚠️ 生产建议**: 建议手动控制，避免自动调优造成的I/O影响

#### 网络协议升级
- **msgr2协议**
  - **首次引入**: Nautilus (v14.x) - 安全性和性能大幅提升
  - **稳定推荐**: Octopus (v15.x) - 修复兼容性问题，可大规模部署
  - **技术价值**: 端到端加密，更强身份验证，更低延迟

#### 编排管理变革
- **cephadm容器化编排**
  - **首次引入**: Octopus (v15.x) - 技术预览版
  - **稳定推荐**: Pacific (v16.x) - 生产可用，功能完善
  - **完全成熟**: Quincy (v17.x) - 替代ceph-deploy，成为主流方案
  - **革命意义**: 从包管理转向容器化，大幅简化运维

#### 分布式文件系统演进
- **CephFS多文件系统**
  - **首次引入**: Nautilus (v14.x) - 基础多FS支持
  - **功能完善**: Octopus (v15.x) - 快照和镜像功能
  - **生产推荐**: Pacific (v16.x) - 镜像功能稳定，可用于生产环境
  - **管理优化**: Squid (v19.x) - Dashboard集成，子卷管理完善

#### 存储引擎发展
- **BlueStore成熟化**
  - **默认引擎**: Mimic (v13.x) - BlueStore替代FileStore成为默认存储引擎
  - **性能优化**: Nautilus (v14.x) - bitmap allocator，大幅性能提升
  - **完全成熟**: Pacific (v16.x) - 性能和稳定性达到最佳
  - **FileStore淘汰**: Reef (v18.x) - FileStore完全移除支持

- **Crimson/Seastore (下一代)**
  - **技术预览**: Quincy (v17.x) - 实验性功能
  - **功能增强**: Squid (v19.x) - 支持RBD工作负载
  - **预期成熟**: 未来版本 - 生产环境可用




## Nautilus (N版本)


### 版本发布情况
- **当前版本**: v14.2.22
- **版本系列**: v14.2.0 到 v14.2.22
- **生命周期**: v14.2.22是Nautilus系列的最后一个版本，EOL

### 主要亮点

#### 中心化配置管理
- **特性**: 相比于之前所有持久化配置都需要写在ceph.conf中的方式，现有的方式更加方便管理，包括查看、修改、审计
- **核心价值**: 统一管理所有集群配置，支持在线修改，提供配置历史和审计功能，大大简化了大规模集群的配置管理

##### 配置管理命令
```bash
# 查看现有的config设置
ceph config dump

# get/set 某一级或是某个实例的配置，支持多级相同配置进行override，使用粒度最小的
ceph config get $level $entityname
ceph config set $level $entityname

# 举例
ceph config set global osd_recovery_sleep_hdd 0.1
ceph config set osd osd_recovery_sleep_hdd 0.1
ceph config set 'osd.*' osd_recovery_sleep_hdd  # 获取服务级别的稍微不一样，需要加*
ceph config set osd.1 osd_recovery_sleep_hdd 0.2

# 查看某个entity的help信息
sudo ceph config help osd_recovery_sleep_hdd

# 修改历史log，做审计
sudo ceph config log
```

##### 配置管理要点
- 有一些之前需要重启生效的参数现在可以在线调节了，可以靠help来确认，比如说osd_target_memory
- osd_class_update_on_start需要在osd创建的时候设置为true，会自动加入到tree中，完成后false，防止osd重启导致crush tree变化

#### PG merging和autotuning
- **⭐ 重大突破**: 这是Ceph历史上的重大突破，Nautilus首次支持PG数量减少(merging)和自动调优功能
- **PG数量减少**: pg数量允许减少，也就是说可以做merge了，解决了之前PG数量只能增加不能减少的限制
- **PG默认autoscale**: pg默认会开启autoscale，自动根据数据量调整PG数量
- **技术意义**: 让存储池管理更加智能化和灵活，减少手动调优工作量
- **⚠️ 生产环境注意**: 调整会导致peering过程，造成I/O卡顿和大量数据迁移。生产环境建议设置 `osd_pool_default_pg_autoscale_mode=off`，手动控制PG调整时机

#### 设备管理和故障预测
- **设备管理**: 新的设备管理功能
- **故障预测**: 基于设备健康指标的故障预测模型

#### 崩溃信息收集
- **服务**: 每个节点上都运行了一个ceph-crash服务
- **功能**: 如果有进程崩溃（比如assert），会收集环境信息和backtrace，并上报

#### 消息传递(msgr2)和安全性配置
- **⭐ 协议革新**: Nautilus首次引入msgr2协议，这是Ceph网络通信的重大升级
- **msgr2协议**: 新的消息传递协议v2，提供更好的安全性和性能
  - **加密传输**: 支持端到端加密，保护集群内部通信安全
  - **身份验证**: 提供更强的身份验证机制，防止中间人攻击
  - **性能提升**: 优化的网络协议栈，降低延迟和CPU开销
- **向后兼容**: 为保证与旧客户端兼容，需要配置同时支持msgr1和msgr2
- **主要配置**: global_id和不启用msgr2（如需兼容旧环境）
- **迁移建议**: 新部署建议直接使用msgr2，旧集群可逐步迁移

#### RADOS相关改进
- **bluestore**: 新的 'bitmap' allocator（比avl allocator性能更好，减少碎片）
- **异步恢复**: 改良的async recovery（减少恢复对客户端I/O的影响）
- **scrub**: scrub支持autorepair（自动修复检测到的数据不一致问题）
- **纠删码**: 添加了新的纠删码算法（提供更好的存储效率和容错能力）
- **pool配额**: pool支持设置quota（限制存储池的空间和对象数量使用）
- **负载均衡**: ceph balancer（自动优化数据分布，提高集群性能均衡性）

#### RBD相关改进
- **命名空间**: 支持namespace（命名空间），实现多租户资源隔离，不同租户可以使用相同的镜像名称而不冲突
- **时间戳**: 卷支持了Creation, access, modification timestamps，便于审计和管理
- **监控增强**: mgr的Prometheus插件现在可以监控单个卷的详细I/O统计信息，提供更精细的性能分析能力

#### CephFS相关改进
- **多文件系统**: 支持在单个集群中创建和管理多个独立的文件系统，每个文件系统有独立的元数据池
  - **⭐ 首次引入**: Nautilus版本首次引入多文件系统支持，为后续版本的完善奠定基础
  - **稳定建议**: 虽然功能可用，但建议在生产环境中先进行充分测试
- **管理接口**: 引入卷管理概念，将复杂的文件系统操作抽象为简单的卷管理，模拟块存储的管理方式
  - **概念映射**: fs(文件系统) → volume(卷)，directory(目录) → subvolume(子卷)
- **稳定性提升**: 
  - **Client caps限制**: 限制了客户端获取caps的速度，防止MDS因过多caps请求而内存溢出(OOM)
  - **Caps**: 是CephFS的能力授权机制，控制客户端对文件/目录的访问权限
- **启动优化**: 
  - **OpenTable文件**: 记录已打开文件的信息，如果MDS重启过慢，可以删除此文件加速启动
  - **适用场景**: 在紧急恢复或大量文件打开的环境中特别有用
- **配额功能**: 内核客户端现在支持配额限制，可以对目录或子卷设置空间和文件数量限制

##### Subvolume
- **子卷管理**: 支持在文件系统内创建隔离的子卷，每个子卷有独立的配额和访问控制
- **CSI集成**: 如果要使用Kubernetes CSI驱动，需要创建专用的CSI子卷组来管理动态卷供应

##### LAZYIO
- **延迟IO模式**: 新的IO模式，允许应用程序延迟写入操作，提高小文件写入性能
- **缓存机制**: 客户端可以缓存写入数据，减少网络往返次数
- **适用场景**: 特别适合频繁小文件写入的工作负载

#### ceph-mgr
- **Prometheus插件**: 
  - **标准监控**: mgr的prometheus插件提供标准的监控指标接口，需要手动启用
  - **集成优势**: 与Grafana等监控系统无缝集成，提供丰富的性能仪表板
- **稳定性问题**: 
  - **CephFS场景**: 在CephFS环境下，mgr可能出现死锁问题，需要定期重启mgr服务
  - **监控建议**: 建议监控mgr服务状态，出现异常及时重启
- **模块管理**: 
  - **Restful模块Bug**: restful模块存在内存泄露问题，建议关闭该模块避免mgr OOM
  - **禁用命令**: `ceph mgr module disable restful`

### 重要更新和修复

#### 性能优化
- **BlueStore缓存**: `bluefs_buffered_io`默认设置为true，提高元数据密集型工作负载的性能
- **内存管理**: `bluestore_cache_trim_max_skip_pinned`默认值增加到1000，控制onodes导致的内存增长
- **监控器修剪**: 引入动态修剪逻辑，通过`paxos_service_trim_max_multiplier`参数控制

#### 稳定性改进
- **BlueStore修复**: 修复了Avl/Hybrid分配器中的意外ENOSPC bug
- **客户端互操作性**: 修复了阻止32位和64位客户端在msgr v2下互操作的长期bug
- **OSD停机**: 新增`--max <n>`选项用于`osd ok-to-stop`命令，提供可以同时停止的OSD数量

#### 新功能
- **⭐ OSD关机通知**: 首次引入`osd_fast_shutdown_notify_mon`选项，允许OSD在快速关机时通知监控器
  - **技术价值**: 改善集群管理，减少不必要的OSD故障告警
  - **后续完善**: 该特性在Pacific版本中得到进一步优化
- **多架构支持**: 修复了armv7l (armhf)与x86_64或aarch64服务器混合部署的问题

#### 部署说明
- **工具**: 目前还是使用ceph-deploy来部署，这是ceph-deploy支持的最后一个版本
- **要求**: ceph-deploy需要python2.7并且需要修改部分代码以支持debian11



---

## Octopus


### 版本发布情况
- **当前版本**: v15.2.17
- **版本系列**: v15.2.0 到 v15.2.17
- **生命周期**: v15.2.17是Octopus系列的最后一个版本，已归档

### 核心主题
Ceph Octopus 版本专注于五个不同的主题：多站点使用、质量、性能、可用性和生态系统。

### 主要亮点

#### 多站点功能
- **CephFS快照**: 
  - **快照调度**: 自动定时创建快照，支持按小时、天、周、月等不同周期
  - **快照修剪**: 自动清理过期快照，避免存储空间无限增长
  - **定期快照自动化**: 通过cron表达式灵活配置快照策略
- **远程同步**: 
  - **跨集群同步**: 支持将数据同步到远程Ceph集群，实现异地备份
  - **增量同步**: 只同步变更的数据，大大减少网络带宽占用
- **RBD镜像**: 
  - **快照镜像模式**: 新增基于快照的镜像模式，提供更可靠的数据一致性
  - **异步复制**: 支持异步复制，减少对主集群性能的影响
- **应用价值**: 
  - **自动备份**: 无需人工干预的自动化备份策略
  - **存储优化**: 增量备份大幅节省存储空间
  - **数据保护**: 提供多层次的数据保护和灾难恢复能力

#### 质量改进
- **崩溃警报**: Ceph守护进程崩溃时会发出简单的健康警报，并可以触发邮件发送
- **设备健康**: 新的"设备"遥测通道改进了设备故障预测模型
- **遥测**: 用户可以选择加入遥测内容扩展

#### 性能优化
- **恢复优化**: 恢复尾部延迟已得到改善，可以通过仅复制对象的增量在恢复期间进行对象同步
- **BlueStore改进**: 
  - 改进了按池计算"omap"（键/值）对象数据
  - 改进了缓存内存管理
  - 减少了SSD设备的分配单元大小

#### 可用性提升
- **⭐ cephadm编排器**: 
  - **🎯 首次引入**: Octopus版本首次引入cephadm编排器，标志着Ceph运维管理的重大革命性变化
  - **管理革命**: 从传统基于包管理转向现代容器化编排管理，大大降低部署和运维复杂度
  - **容器化部署**: 所有Ceph组件都运行在容器中，提供更好的隔离性和可维护性
  - **自动化管理**: 自动处理服务发现、健康检查、故障恢复等运维任务
  - **成熟度**: 在Octopus中为技术预览版，Pacific版本后达到生产可用成熟度
- **编排器API**: 
  - **统一接口**: 提供统一的API接口与cephadm编排器交互
  - **声明式管理**: 支持声明式的服务配置，简化集群管理
- **SSH管理**: 
  - **无代理架构**: 通过SSH连接管理各个节点，无需在每个节点安装额外代理
  - **安全连接**: 使用SSH密钥认证，确保管理连接的安全性
  - **命令执行**: 支持在远程节点上执行管理命令，实现集中化管理
- **仪表板集成**: 
  - **可视化管理**: Ceph Dashboard与编排器深度集成，提供图形化的集群管理界面
  - **服务管理**: 可以通过Web界面直接管理服务的启停、升级等操作
- **健康警报**: 
  - **警报管理**: 支持暂时或永久静音特定的健康警报
  - **降噪功能**: 避免非关键警报干扰，专注于真正重要的问题
- **历史意义**: cephadm标志着Ceph从传统的基于包管理的部署方式，向现代容器化编排管理的重大转变，大大降低了Ceph集群的部署、升级和日常运维复杂度

#### 生态系统增强
- **ceph-csi**: 现在通过RBD和CephFS支持RWO和RWX
- **Rook集成**: 与Rook的集成默认提供交钥匙ceph-csi
- **功能支持**: 包括RBD镜像、RGW多站点，最终还将包括CephFS镜像

### 重要修复和安全更新

#### 安全漏洞修复
- **CVE-2022-0670**: 修复了OpenStack Manila Native-CephFS的路径限制绕过漏洞
- **影响范围**: 仅影响使用OpenStack Manila提供native CephFS访问的集群
- **修复内容**: 修复了Ceph Manager中"volumes"插件的bug，该插件负责管理CephFS子卷

#### 关键Bug修复
- **SnapMapper key格式**: 修复了v15.2.x中引入的SnapMapper key格式转换bug
- **RBD性能**: 修复了`rbd perf image iostat`和`rbd perf image iotop`命令的功能
- **稳定性**: 多项稳定性和性能改进


---

## Pacific


### 版本发布情况
- **当前版本**: v16.2.15
- **版本系列**: v16.2.0 到 v16.2.15
- **生命周期**: v16.2.15是Pacific系列的最后一个版本，已归档

### 主要亮点

#### 管理简化
- **配置异常检测**: 新增检测配置异常的能力，帮助识别和解决配置问题
- **集群升级监控**: 新增健康检查`DAEMON_OLD_VERSION`，当不同版本的Ceph在守护进程上运行时发出警告
- **进度模块**: MGR progress模块现在可以通过`ceph progress on`和`ceph progress off`命令开关

#### 存储后端优化
- **ceph-volume重写**: `lvm batch`子命令进行了重大重写，修复了许多错误并改善了可用性
- **OSD改进**: 
  - 新增`osd_compact_on_start`配置选项，在启动时触发OSD压缩（压缩存储引擎数据库以减少空间占用）
  - 改进了OSD快速关闭机制，基于Nautilus引入的`osd_fast_shutdown_notify_mon`特性进行优化，提升与monitor的通信效率
- **BlueStore增强**: 多项BlueStore改进和性能优化

#### 调度器改进
- **mclock调度器**: 
  - **QoS调度**: 经过精炼的mclock调度器，基于mClock算法实现精确的服务质量控制
  - **工作原理**: 通过预留(reservation)、限制(limitation)和权重(weight)三个参数，精确控制不同类型操作的资源分配
  - **内置配置文件**: 提供三种预设配置，用户可根据业务需求选择：
    - **high_client_ops** (默认): 为外部客户端操作分配更多OSD带宽，优先保证业务I/O性能
    - **high_recovery_ops**: 优化恢复操作，加快数据重建和修复速度，适合维护窗口使用
    - **balanced**: 平衡配置，在客户端I/O和后台操作之间保持均衡
  - **技术优势**: 相比传统的WRR(加权轮询)调度器，mclock能提供更精确的性能隔离和QoS保证
- **负载均衡器**: 
  - **upmap模式**: 现在默认在upmap模式下启用，提供更精确的PG映射控制
  - **客户端要求**: 要求设置`require_min_compat_client luminous`，确保客户端支持upmap功能
  - **优势**: upmap模式比传统的CRUSH reweight方式能实现更精确的数据分布均衡

#### 安全改进
- **身份验证协议**: cephx认证协议版本2 (`CEPHX_V2`)现在默认必需
- **重放攻击保护**: 为授权者添加了重放攻击保护，增强了msgr v1消息签名

#### RBD改进
- **快速差异模式**: 当对时间起点进行差异比较时，在快速差异模式下显著提升性能
- **缓存优化**: 共享只读父缓存的配置选项`immutable_object_cache_watermark`更新为0.9
- **克隆功能**: 支持从非用户类型快照进行克隆
- **rbd-wnbd驱动**: 获得了多路复用映像映射的能力，提高了Windows环境下的资源使用效率

#### RGW改进
- **持久化Bucket通知**: 持久化Bucket通知深度整合，提供更可靠的通知机制
- **AWS兼容API**: 添加了"GetTopicAttributes" API替代现有的"GetTopic" API
- **日志管理**: `bilog-trim`、`datalog-trim`、`sync-error-trim`现在只接受标记范围的单个标记

#### CephFS改进
- **🎯 镜像功能完善**: CephFS镜像功能增强，从Nautilus的基础支持发展为生产可用的解决方案
  - **改进同步机制**: 优化的增量同步算法，大幅减少网络带宽占用
  - **生产可用**: 相比Octopus的技术预览版本，Pacific版本达到生产环境部署标准
- **会话管理**: MDS会驱逐没有推进请求tid的客户端，防止会话元数据大量积累
- **灾备方案优化**: 
  - **定时快照**: 增量更新机制，避免全量备份的开销
  - **智能同步**: `cephfs-mirror`守护进程依靠readdir diff识别目录树中的更改
  - **增量同步**: diff应用于远程文件系统中的目录，仅同步两个快照之间已更改的文件
  - **运维价值**: 为企业级CephFS提供可靠的异地灾备解决方案

### 重要变更

#### 术语变更
- **blacklist → blocklist**: 所有相关命令和配置选项都已更新
- **命令变更**:
  ```bash
  # 旧命令 → 新命令
  ceph osd blacklist → ceph osd blocklist
  ceph <tell|daemon> osd.<NNN> dump_blacklist → ceph <tell|daemon> osd.<NNN> dump_blocklist
  ```

#### 配置变更
- **scrub时间配置**: 
  - `osd_scrub_begin_hour`和`osd_scrub_end_hour`合法值为0-23
  - `osd_scrub_begin_week_day`和`osd_scrub_end_week_day`合法值为0-6
- **监控器配置**: 新增`mon_allow_pool_size_one`选项，默认禁用
- **池大小设置**: 设置池大小为1时需要`--yes-i-really-mean-it`标志

#### API变更
- **librados API**: 
  - `rados_blacklist_add` → `rados_blocklist_add`
  - `get_pool_is_selfmanaged_snaps_mode` 已被弃用，建议使用`pool_is_in_selfmanaged_snaps_mode`



### 重要注意事项
- **升级路径**: 必须先升级到Octopus (15.2.z)才能升级到Pacific
- **客户端兼容性**: 新集群默认只支持Luminous及更新的客户端
- **身份验证**: 默认要求cephx v2协议，旧客户端需要设置配置选项
- **NFS Ganesha**: 升级前需要删除现有的nfs-ganesha集群，升级后重新部署
- **术语变更**: 所有blacklist相关术语已更改为blocklist
- **安全升级**: 建议尽快升级以获得最新的安全修复

---

## Quincy


### 版本发布情况
- **当前版本**: v17.2.9
- **版本系列**: v17.2.0 到 v17.2.9
- **生命周期**: 已达到生命周期结束（EOL），v17.2.9是最后一个修复版本，已归档

### 主要亮点

#### Dashboard
- **界面改进**: Dashboard界面改进，提供更好的用户体验
- **功能增强**: 更多的管理功能和监控能力

#### RADOS
- **🎯 mclock调度器成熟**: mclock调度器成为默认选项，相比Pacific版本的引入，Quincy版本实现了生产环境的全面部署
  - **服务质量保证**: 为Ceph客户端提供相对于后台操作(如recovery、backfill)的服务质量优先级
  - **智能调度**: 根据I/O类型和优先级动态分配资源，确保关键业务不受后台任务影响
  - **配置简化**: 提供预设配置模板，降低调优复杂度
- **性能优化**: 各种性能改进，提升整体集群性能
- **API变更**: `get_pool_is_selfmanaged_snaps_mode` C++ API已被弃用，建议使用更安全的`pool_is_in_selfmanaged_snaps_mode`替代

#### RBD
- **快速差异模式**: 当对时间起点进行差异比较时，在快速差异模式下显著提升性能
- **命令增强**: 
  - `rbd children`命令添加了`--image-id`选项，可以用于垃圾箱中的镜像
  - Python绑定中暴露了`RBD_IMAGE_OPTION_CLONE_FORMAT`选项
  - Python绑定中暴露了`RBD_IMAGE_OPTION_FLATTEN`选项
- **性能优化**: 为QEMU实时磁盘同步和备份用例带来了显著的性能改进

#### CephFS
- **文件系统ID**: 可以指定ID ("fscid")创建文件系统，这在某些恢复场景中非常有用
- **恢复场景**: 例如监控数据库丢失和重建，恢复的文件系统可以拥有与之前相同的ID
- **文件系统重命名**: 支持使用`fs rename`命令重命名文件系统
- **MDS升级**: MDS升级不再需要在升级文件系统的唯一活动MDS之前停止所有备用MDS守护进程
- **故障处理**: 备用重放守护进程无法重放日志时，现在会将等级标记为"damaged"

#### 其他功能
- **SQLite支持**: 新库libcephsqlite现已推出
- **虚拟文件系统**: 在RADOS之上提供SQLite虚拟文件系统(VFS)
- **数据条带化**: 数据库和日志通过RADOS跨多个对象进行条带化
- **可扩展性**: 几乎可以无限扩展，吞吐量仅受SQLite客户端限制
- **LevelDB迁移**: 要求从LevelDB迁移到RocksDB（如果适用）

---

## Reef

Reef是Ceph的第18个稳定版本，以reef squid (Sepioteuthis)命名。

### 版本发布情况
- **当前版本**: v18.2.7 (推荐所有用户升级)
- **版本系列**: v18.2.0 到 v18.2.7
- **生命周期**: 当前活跃的稳定版本
- **⚠️ 关键修复**: v18.2.7修复了18.2.5和18.2.6版本中的严重BlueStore回归问题
  - **问题描述**: BlueStore在处理大型对象(>4MB)时出现数据完整性问题，可能导致对象读取错误或损坏
  - **影响范围**: 主要影响RGW大文件上传、RBD大块写入等场景，可能引起静默数据损坏
  - **回归原因**: v18.2.5中引入的BlueStore压缩优化代码存在边界条件处理错误
  - **修复措施**: 回滚了有问题的压缩优化，并增加了额外的数据完整性校验
  - **升级建议**: 如果正在使用v18.2.5或v18.2.6，强烈建议立即升级到v18.2.7

### 主要亮点

#### 主要功能
- **FileStore弃用**: FileStore在Reef中不再受支持，用户需要迁移到BlueStore
- **RocksDB升级**: RocksDB已升级到版本7.9.2，性能有显著改进
- **命令更新**: `perf dump`和`perf schema`命令已被弃用，取而代之的是新的`counter dump`和`counter schema`命令
- **缓存分层弃用**: 缓存分层现在已被弃用
- **读取平衡器**: 新功能"读取平衡器"现已可用，允许用户在集群上平衡每个池的主PG
- **新命令**: 添加了`ceph osd rm-pg-upmap-primary-all`命令，允许用户清除osdmap中的所有pg-upmap-primary映射

#### RGW
- **存储桶重新分片**: 多站点配置现在支持存储桶重新分片
- **多站点复制**: 多站点复制的稳定性和一致性有显著改进
- **加密压缩**: 现在支持使用服务器端加密上传的对象的压缩
- **安全考虑**: 在启用`compress-encrypted`区域组功能之前，必须将所有区域升级到Reef

#### Dashboard
- **新页面**: 有一个改进布局的新Dashboard页面
- **卡片显示**: 活动警报和一些重要图表现在显示在卡片内

#### RBD
- **分层加密**: 添加了对分层客户端加密的支持
- **NBD映射**: `try-netlink`映射选项现在成为默认选项并已弃用，如果内核不支持NBD netlink接口，则会使用传统ioctl接口重试

#### 遥测
- **排行榜**: 用户现在可以选择参与遥测公共仪表板中的排行榜 ([公共仪表板](https://telemetry-public.ceph.com/))
- **集群描述**: 用户可以提供集群描述，将在排行榜中公开显示
- **相关命令**:
  ```bash
  # 查看遥测预览
  ceph telemetry preview
  
  # 开启遥测
  ceph telemetry on
  
  # 开启排行榜
  ceph config set mgr mgr/telemetry/leaderboard true
  
  # 添加排行榜描述
  ceph config set mgr mgr/telemetry/leaderboard_description 'Cluster description'
  ```


---

## Squid

Squid是Ceph的第19个稳定版本，是当前活跃的稳定版本。

### 版本发布情况
- **当前版本**: v19.2.2 (推荐所有用户升级)
- **版本系列**: v19.2.0 到 v19.2.x
- **生命周期**: 当前活跃的稳定版本
- **重要修复**: v19.2.2修复了RGW CopyObject数据丢失的关键bug
- **技术预览**: Crimson/Seastore技术预览版本，支持RBD工作负载在复制池上运行

### 主要亮点

#### Dashboard界面改进
- **重新设计的导航布局**: 导航布局已重新组织，提升可用性和关键功能的访问便利性
- **CephFS快照和克隆管理**: 支持管理CephFS快照和克隆，以及快照调度管理
- **授权管理**: 管理CephFS资源的授权能力
- **挂载助手**: 提供CephFS卷挂载的帮助工具

#### 核心功能改进
- **CephFS子卷标记**: `fs subvolume create`命令现在支持通过`--earmark`选项为子卷添加标记
- **扩展属性**: 扩展了CephFS虚拟扩展属性的removexattr支持
- **用户账户**: RGW用户账户功能解锁了多个AWS兼容的IAM API，用于自服务管理用户、密钥、组、角色、策略等

#### RADOS改进
- **平衡器性能**: 修复了平衡器mgr模块的性能瓶颈
- **HDD配置优化**: 基于大规模测试，优化了HDD OSD的默认分片配置：
  - `osd_op_num_shards_hdd = 1` (原来是5)
  - `osd_op_num_threads_per_shard_hdd = 5` (原来是1)
- **BlueStore优化**: BlueStore针对快照密集型工作负载进行了优化
- **RocksDB压缩**: BlueStore RocksDB LZ4压缩现在默认启用，提高平均性能和"快速设备"空间使用
- **CRUSH规则**: 新的CRUSH规则类型MSR（多步重试）允许更灵活的EC配置

#### 管理器模块改进
- **REST模块**: REST管理器模块现在会根据`max_requests`选项修剪请求，防止内存不足(OOM)问题

#### RGW增强
- **Topic所有权**: SNS topics现在有所有者概念，默认只有所有者可以读写
- **Topic策略**: 支持topic策略文档来授予权限给其他用户
- **持久化通知**: 修复了持久化通知中topic参数修改的问题
- **用户ID**: 在bucket通知中，`ownerIdentity`内的`principalId`现在包含完整的用户ID
- **版本化bucket**: 新增工具来识别和修正版本化bucket索引的问题

#### 遥测改进
- **基本频道**: 遥测的`basic`频道现在捕获池标志，帮助了解功能采用情况
- **遥测命令**: 使用`ceph telemetry on`可以选择加入遥测

### 重要修复

#### 数据安全修复
- **RGW数据丢失**: 修复了关键的CopyObject复制对象到自身时的数据丢失bug（v19.2.2中修复）
- **垃圾收集**: 解决了尾部对象被错误标记为垃圾收集的问题
- **恢复工具**: 提供了实验性的`rgw-gap-list`工具来识别损坏的对象
- **重要安全修复**: 修复了可能导致数据丢失的关键安全问题

#### 性能和稳定性
- **内核设备**: 一系列针对内核设备discard的优化
- **消息传递**: 修复了AsyncMessenger中的连接计数问题
- **BlueStore**: 多项BlueStore改进和bug修复
- **平衡器性能**: 修复了平衡器mgr模块的性能瓶颈
- **HDD配置优化**: 优化了HDD OSD的默认分片配置以提高mClock调度性能

---




## 参考资料

### 官方文档
- [Ceph官方文档](https://docs.ceph.com/)
- [Ceph版本发布说明](https://docs.ceph.com/en/latest/releases/)
- [Ceph开发者文档](https://docs.ceph.com/en/latest/dev/)

### 技术文章
- [New in Mimic: Centralized Configuration Management](https://ceph.io/en/news/blog/2018/new-mimic-centralized-configuration-management/)
- [New in Nautilus: Crash Dump Telemetry](https://ceph.io/en/news/blog/2019/new-in-nautilus-crash-dump-telemetry/)
- [New in Octopus: Dashboard Features](https://ceph.io/en/news/blog/2020/new-in-octopus-dashboard-features/)

### 技术讲座
- Ceph Tech Talk - What's New In Octopus
- Ceph Developer Summit Quincy: RADOS Follow-up
- Diving Deep with Squid



## 特别说明

本文仅供参考，如果有错误的地方，请指正
