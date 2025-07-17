## MDS 系统架构概览
Ceph MDS是CephFS (Ceph File System) 的核心组件，负责处理所有文件系统元数据操作。MDS的设计采用分布式、可扩展的架构，支持多活MDS和动态负载均衡。
### MDS在Ceph生态中的定位
```mermaid
graph TB
    Client[CephFS Client] --> MDS[MDS Cluster]
    MDS --> RADOS[RADOS Storage Layer]
    MDS --> Mon[Monitor Cluster]
    subgraph "MDS In Ceph"
        subgraph "Client Layer"
            Client
            Fuse[FUSE Client]
            Kernel[Kernel Client]
        end
        subgraph "Metadata Layer"
            MDS
            MDSStandby[Standby MDS]
            MDSActive[Active MDS]
        end
        subgraph "Storage Layer"
            RADOS
            OSD[OSD Cluster]
            Pool[Metadata Pool]
        end
        subgraph "Management Layer"
            Mon
            Mgr[Manager]
        end
    end
    Client -.-> Fuse
    Client -.-> Kernel
    MDS --> MDSStandby
    MDS --> MDSActive
    RADOS --> OSD
    RADOS --> Pool
    Mon --> Mgr
```
### MDS 模块架构
![image.png|600](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images20250530152434.png)
## MDS 核心子模块架构分析
### MDSMap 管理模块
MDSMap是MDS集群状态管理的核心模块，负责维护MDS集群的拓扑信息和状态。
```mermaid
graph LR
    subgraph "MDSMap Module"
        MDSMap[MDSMap Manager]
        ActiveMDS[Active MDS List]
        StandbyMDS[Standby MDS List]
        FSMap[Filesystem Map]
        MDSMap --> ActiveMDS
        MDSMap --> StandbyMDS
        MDSMap --> FSMap
    end
    Monitor[Monitor Cluster] --> MDSMap
    MDSMap --> ClientView[Client View]
    MDSMap --> MDSInstances[MDS Instances]
```
**核心功能：**
- MDS集群成员管理
- Rank分配和故障转移
- 文件系统到MDS的映射
- 状态同步和版本控制
**关键配置参数：**
```bash
# 设置最大活动MDS数量
ceph fs set <fsname> max_mds <count>
# 查看MDS状态
ceph mds stat
ceph fs status <fsname>
```
### Session管理模块
Session管理模块负责处理客户端连接和会话状态维护。
```mermaid
graph TB
    subgraph "Session Management"
        SessionMgr[Session Manager]
        ClientSessions[Client Sessions]
        SessionCache[Session Cache]
        Capabilities[Capability Grants]
        SessionMgr --> ClientSessions
        SessionMgr --> SessionCache
        SessionMgr --> Capabilities
        subgraph "Session State"
            Opening[Opening]
            Open[Open]
            Stale[Stale]
            Killing[Killing]
        end
        ClientSessions --> Opening
        ClientSessions --> Open
        ClientSessions --> Stale
        ClientSessions --> Killing
    end
    Clients[CephFS Clients] --> SessionMgr
    Monitor[Monitor] --> SessionMgr
    MDCache[MDS Cache] --> Capabilities
```
**核心功能：**
- 客户端连接认证
- 会话生命周期管理
- Capability分发和回收
- 会话超时处理
### MDCache模块
MDCache是MDS的核心缓存模块，负责元数据的缓存、一致性维护和分布式协调。
```mermaid
graph TB
    subgraph "MDCache Architecture"
        MDCache[MD Cache Manager]
        subgraph "Cache Structures"
            Inodes[Inode Cache]
            Dentries[Dentry Cache]
            Dirfrags[Dirfrag Cache]
        end
        subgraph "Cache Operations"
            Fetch[Fetch Operations]
            Discover[Discovery]
            Migration[Cache Migration]
            Trim[Cache Trimming]
        end
        subgraph "Coherency"
            Locks[Distributed Locks]
            Caps[Capabilities]
            Leases[Client Leases]
        end
        MDCache --> Inodes
        MDCache --> Dentries
        MDCache --> Dirfrags
        MDCache --> Fetch
        MDCache --> Discover
        MDCache --> Migration
        MDCache --> Trim
        MDCache --> Locks
        MDCache --> Caps
        MDCache --> Leases
    end
    RADOS[RADOS Storage] --> Fetch
    Clients[CephFS Clients] --> Caps
    OtherMDS[Other MDS] --> Migration
```
**核心功能：**
- 分布式元数据缓存
- 缓存一致性协议
- 元数据预取和预测性缓存
- 内存管理和LRU淘汰
### MDS Balancer模块
负载均衡模块确保元数据负载在多个活动MDS之间均匀分布。
```mermaid
graph LR
    subgraph "MDS Balancer"
        Balancer[Load Balancer]
        subgraph "Metrics Collection"
            CPUMetrics[CPU Usage]
            MemMetrics[Memory Usage]
            IOMetrics[I/O Metrics]
            ClientLoad[Client Load]
        end
        subgraph "Balancing Strategies"
            HotSpot[Hot Spot Detection]
            Migration[Directory Migration]
            Fragmentation[Dir Fragmentation]
        end
        Balancer --> CPUMetrics
        Balancer --> MemMetrics
        Balancer --> IOMetrics
        Balancer --> ClientLoad
        Balancer --> HotSpot
        Balancer --> Migration
        Balancer --> Fragmentation
    end
    MDSRanks[MDS Ranks] --> Balancer
    MDCache --> Migration
    Monitor --> Balancer
```
**核心配置：**
```bash
# 启用MDS负载均衡
ceph config set mds mds_bal_mode 2
# 设置负载均衡阈值
ceph config set mds mds_bal_need_min 0.2
ceph config set mds mds_bal_need_max 1.25
```
### 日志和恢复模块
MDS Journal模块负责元数据操作的持久化日志记录和故障恢复。
```mermaid
graph TB
    subgraph "Journal & Recovery"
        Journal[MDS Journal]
        subgraph "Journal Operations"
            LogEvents[Log Events]
            Segments[Journal Segments]
            Trimming[Log Trimming]
        end
        subgraph "Recovery Process"
            Replay[Journal Replay]
            Resolution[Conflict Resolution]
            Cleanup[Recovery Cleanup]
        end
        subgraph "Storage Backend"
            MDSPool[Metadata Pool]
            Objects[Journal Objects]
            RADOS_Journal[RADOS Backend]
        end
        Journal --> LogEvents
        Journal --> Segments
        Journal --> Trimming
        Journal --> Replay
        Journal --> Resolution
        Journal --> Cleanup
        Journal --> MDSPool
        Journal --> Objects
        Journal --> RADOS_Journal
    end
    MDSOperations[MDS Operations] --> LogEvents
    StandbyMDS[Standby MDS] --> Replay
    Monitor --> Resolution
```
## MDS状态机和生命周期
### MDS状态转换图
```mermaid
stateDiagram-v2
    [*] --> boot
    boot --> standby
    standby --> standby_replay
    standby_replay --> resolve
    resolve --> reconnect
    reconnect --> rejoin
    rejoin --> active
    active --> stopping
    stopping --> [*]
    active --> resolve : failover
    standby --> resolve : takeover
    active --> standby : rank_stop
    note right of active
        正常工作状态
        处理客户端请求
    end note
    note right of standby
        待机状态
        等待分配rank
    end note
    note right of resolve
        解决元数据冲突
        处理分布式状态
    end note
```
### MDS启动和初始化流程
```mermaid
sequenceDiagram
    participant Monitor as Monitor
    participant MDS as MDS Daemon  
    participant RADOS as RADOS
    participant Standby as Standby MDS
    MDS->>Monitor: 注册为standby
    Monitor->>MDS: 分配rank
    MDS->>RADOS: 读取journal
    MDS->>MDS: 回放journal
    MDS->>Monitor: 报告active状态
    Monitor->>Standby: 通知状态变化
    MDS->>MDS: 开始处理客户端请求
```
## MDS与上下游组件关系
### MDS与Monitor的交互
```mermaid
graph LR
    subgraph "MDS-Monitor Interaction"
        MDS[MDS Daemon]
        Monitor[Monitor Cluster]
        subgraph "Information Exchange"
            MDSMap_Update[MDSMap Updates]
            Health_Report[Health Reports]
            Beacon[MDS Beacon]
            Commands[Admin Commands]
        end
        MDS --> MDSMap_Update
        MDS --> Health_Report
        MDS --> Beacon
        Monitor --> Commands
        MDSMap_Update --> Monitor
        Health_Report --> Monitor
        Beacon --> Monitor
        Commands --> MDS
    end
```
**关键监控命令：**
```bash
# 查看MDS健康状态
ceph health detail
# 检查慢速元数据IO
ceph mds perf dump
# 查看MDS告警
ceph health mute MDS_SLOW_REQUEST
# MDS性能统计
ceph daemon mds.<name> perf dump
```
### MDS与RADOS的交互
MDS通过RADOS存储所有持久化元数据：
```mermaid
graph TB
    subgraph "MDS-RADOS Interaction"
        MDS[MDS Daemon]
        subgraph "RADOS Operations"
            MetadataIO[Metadata I/O]
            JournalIO[Journal I/O]
            BackingStore[Backing Store Ops]
        end
        subgraph "RADOS Pools"
            MetadataPool[Metadata Pool]
            DataPool[Data Pool]
        end
        MDS --> MetadataIO
        MDS --> JournalIO
        MDS --> BackingStore
        MetadataIO --> MetadataPool
        JournalIO --> MetadataPool
        BackingStore --> DataPool
    end
    LIBRADOS[librados] --> MetadataPool
    LIBRADOS --> DataPool
```
### MDS与客户端的交互
```mermaid
sequenceDiagram
    participant Client as CephFS Client
    participant MDS as MDS Active
    participant RADOS as RADOS
    Client->>MDS: 建立session
    MDS->>Client: 分发capabilities
    Client->>MDS: 元数据请求(open/mkdir/stat)
    MDS->>RADOS: 读取/更新元数据
    RADOS->>MDS: 返回结果
    MDS->>Client: 返回元数据
    Client->>MDS: 释放capabilities
    MDS->>Client: 确认释放
```
## MDS常见故障诊断和恢复
```bash
# 检查MDS状态
ceph fs status
ceph mds stat
# MDS性能分析
ceph daemon mds.<name> perf dump
ceph daemon mds.<name> dump cache
# 检查客户端连接
ceph daemon mds.<name> session ls
```
## MDS监控和告警
### 关键性能指标
```bash
# MDS性能监控
ceph daemonperf mds
# 关键指标：
# - mds.inodes: 缓存的inode数量
# - mds.reply_latency: 响应延迟
# - mds.request_rate: 请求速率
# - mds.sessions: 活动会话数
```
### 告警配置
重要告警项目：
- MDS_SLOW_REQUEST: 慢请求告警
- MDS_SLOW_METADATA_IO: 慢元数据IO
- MDS_INSUFFICIENT_STANDBY: 待机MDS不足
- MDS_HEALTH_READ_ONLY: MDS只读状态
