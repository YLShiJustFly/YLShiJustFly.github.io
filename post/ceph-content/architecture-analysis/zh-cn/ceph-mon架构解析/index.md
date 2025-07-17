# Ceph Monitor 架构解析
## Monitor总体架构概览
### 核心功能定位
Ceph Monitor作为集群的控制平面，主要承担以下核心职责：
- **集群映射维护**：管理MonitorMap、OSDMap、CRUSHMap、MDSMap、PGMap等关键映射信息
- **状态监控与健康检查**：实时监控集群状态、生成健康报告
- **分布式一致性保证**：基于Paxos算法确保集群元数据的一致性
- **认证与授权**：管理CephX认证系统和用户权限
- **选举与仲裁**：维护Monitor quorum，处理故障恢复
### Monitor架构图
```mermaid
graph TB
    subgraph "Ceph Monitor 核心架构"
        A[Monitor Daemon] --> B[MonitorStore]
        A --> C[Paxos Engine]
        A --> D[Election Module]
        A --> E[Health Module]
        A --> F[Config Module]
        A --> G[Auth Module]
        B --> B1[ClusterMap Storage]
        B --> B2[Configuration DB]
        B --> B3[Transaction Log]
        C --> C1[Proposal Processing]
        C --> C2[Leader Election]
        C --> C3[Consensus Coordination]
        D --> D1[Connectivity Strategy]
        D --> D2[Quorum Management]
        D --> D3[Split-brain Prevention]
        E --> E1[Health Checks]
        E --> E2[Status Reporting]
        E --> E3[Alert Generation]
        F --> F1[Config Key-Value Store]
        F --> F2[Runtime Configuration]
        F --> F3[Config Distribution]
        G --> G1[CephX Authentication]
        G --> G2[User Management]
        G --> G3[Capability Control]
    end
    subgraph "外部交互"
        H[OSD Daemons] --> A
        I[MDS Daemons] --> A
        J[Client Applications] --> A
        K[Admin Tools] --> A
        L[Dashboard/Grafana] --> A
    end
```
---
## Monitor核心子模块分析
### MonitorStore存储引擎
**功能概述**：
MonitorStore是Monitor的持久化存储引擎，基于RocksDB实现，负责存储所有关键的集群元数据。
**核心架构**：
```mermaid
graph LR
    subgraph "MonitorStore Architecture"
        A[MonitorStore Interface] --> B[RocksDB Backend]
        A --> C[Transaction Manager]
        A --> D[Snapshot Management]
        B --> B1[Cluster Maps]
        B --> B2[Configuration Data]
        B --> B3[Authentication Info]
        B --> B4[Health Records]
        C --> C1[ACID Transactions]
        C --> C2[Batch Operations]
        C --> C3[Rollback Support]
        D --> D1[Consistent Snapshots]
        D --> D2[Backup/Restore]
        D --> D3[Recovery Points]
    end
```
### Paxos一致性引擎
**功能概述**：
Paxos引擎是Monitor最核心的模块，实现分布式一致性算法，确保集群元数据在所有Monitor节点间保持一致。
**架构设计**：
```mermaid
graph TB
    subgraph "Paxos Consensus Engine"
        A[Paxos Service] --> B[Proposer Role]
        A --> C[Acceptor Role] 
        A --> D[Learner Role]
        B --> B1[Proposal Generation]
        B --> B2[Phase 1: Prepare]
        B --> B3[Phase 2: Accept]
        C --> C1[Promise Handling]
        C --> C2[Accept Requests]
        C --> C3[Vote Management]
        D --> D1[Value Learning]
        D --> D2[State Synchronization]
        D --> D3[Catchup Mechanism]
        E[Leader Election] --> F[Quorum Management]
        E --> G[Term Management]
        F --> F1[Majority Calculation]
        F --> F2[Split-brain Prevention]
        F --> F3[Network Partition Handling]
    end
```
**核心运行机制**：
1. **提案阶段**：
```bash
# 检查当前quorum状态
ceph quorum_status
```
1. **一致性保证流程**：
   - **Phase 1（准备阶段）**：Leader向所有Acceptor发送Prepare请求
   - **Phase 2（接受阶段）**：收集多数派响应后发送Accept请求
   - **Commit（提交阶段）**：达成一致后向所有节点广播提交消息
### 选举与仲裁模块
**功能概述**：
负责Monitor集群的Leader选举、quorum维护和故障检测，确保集群在各种网络条件下的可用性。
**选举策略架构**：
```mermaid
graph LR
    subgraph "Election & Quorum Module"
        A[Election Manager] --> B[Connectivity Strategy]
        A --> C[Classic Strategy]
        A --> D[Disallowed Strategy]
        B --> B1[Network Reachability]
        B --> B2[Latency Measurement]
        B --> B3[Bandwidth Testing]
        C --> C1[Monitor Rank Based]
        C --> C2[Simple Majority]
        C --> C3[Static Priority]
        E[Quorum Management] --> F[Split-brain Detection]
        E --> G[Failure Detection]
        E --> H[Recovery Coordination]
        F --> F1[Network Partition Handling]
        F --> F2[Minority Protection]
        F --> F3[Quorum Loss Recovery]
    end
```
**关键配置参数**：
```ini
# Monitor选举相关配置
[mon]
mon_election_timeout = 5
mon_lease = 5
mon_lease_renew_interval_factor = 0.6
mon_lease_ack_timeout_factor = 2.0
mon_accept_timeout_factor = 2.0
```
**选举触发条件**：
- Monitor节点启动或重启
- 网络分区或连接断开
- Leader节点故障或无响应
- 手动触发选举（运维操作）
### 健康监控模块
**功能概述**：
实时监控集群各组件的健康状态，生成告警信息，并提供详细的诊断数据。
**监控架构**：
```mermaid
graph TB
    subgraph "Health Monitoring Module"
        A[Health Service] --> B[Health Checks]
        A --> C[Status Aggregation]
        A --> D[Alert Management]
        B --> B1[OSD Health]
        B --> B2[PG Status]
        B --> B3[Cluster Capacity]
        B --> B4[Network Connectivity]
        B --> B5[Daemon Liveness]
        C --> C1[Global Status]
        C --> C2[Component Status]
        C --> C3[Trend Analysis]
        D --> D1[Warning Generation]
        D --> D2[Error Classification]
        D --> D3[Recovery Suggestions]
        E[Metrics Collection] --> F[Performance Data]
        E --> G[Resource Usage]
        E --> H[I/O Statistics]
    end
```
**健康检查命令示例**：
```bash
# 获取集群整体状态
ceph status
ceph -s
# 查看详细健康信息
ceph health detail
# 监控集群状态变化
ceph -w
# 查看特定组件状态
ceph pg stat
ceph osd stat
ceph mon stat
```
**健康状态分类**：
- **HEALTH_OK**：集群运行正常
- **HEALTH_WARN**：存在警告但不影响数据安全
- **HEALTH_ERR**：存在错误需要立即处理
### 配置管理模块
**功能概述**：
管理集群和守护进程的配置参数，支持运行时配置更新和分发。
**配置管理架构**：
```mermaid
graph LR
    subgraph "Configuration Management"
        A[Config Service] --> B[Key-Value Store]
        A --> C[Config Distribution]
        A --> D[Runtime Updates]
        B --> B1[Global Settings]
        B --> B2[Per-Daemon Config]
        B --> B3[User Preferences]
        C --> C1[Push Mechanism]
        C --> C2[Pull Mechanism]
        C --> C3[Change Notifications]
        D --> D1[Hot Reconfiguration]
        D --> D2[Restart Required Flags]
        D --> D3[Validation Checks]
        E[Config Sources] --> F[ceph.conf File]
        E --> G[Command Line]
        E --> H[Monitor Database]
        E --> I[Environment Variables]
    end
```
**配置操作命令**：
```bash
# 设置全局配置
ceph config set global public_network 192.168.160.0/24
# 查看特定守护进程配置
ceph config show osd.0
ceph config show-with-defaults osd.0
# 设置配置键值
ceph config-key set <key> <value>
# 生成最小配置文件
ceph config generate-minimal-conf > /etc/ceph/ceph.conf
```
## Monitor与其他组件的交互关系
### Monitor-OSD交互
**交互模式**：
```mermaid
sequenceDiagram
    participant OSD as OSD Daemon
    participant MON as Monitor
    participant CRUSH as CRUSH Map
    OSD->>MON: Heartbeat & Status Report
    MON->>OSD: OSD Map Updates
    OSD->>MON: PG Status Updates
    MON->>CRUSH: Update CRUSH Rules
    MON->>OSD: New CRUSH Map
    OSD->>MON: Confirm Map Epoch
```
**关键交互内容**：
- **心跳检测**：OSD定期向Monitor报告存活状态
- **状态更新**：PG状态、容量信息、性能指标
- **映射分发**：OSD Map、CRUSH Map更新通知
- **故障处理**：OSD故障检测和标记
### Monitor-MDS交互
**交互架构**：
```mermaid
sequenceDiagram
    participant MDS as MDS Daemon
    participant MON as Monitor
    participant FS as FileSystem
    MDS->>MON: Register & Get MDS Map
    MON->>MDS: Assign MDS Rank
    MDS->>MON: Status & Metadata Updates
    MON->>MDS: FileSystem Configuration
    MDS->>MON: Load Balancing Metrics
    MON->>MDS: Rank Reassignment
```
