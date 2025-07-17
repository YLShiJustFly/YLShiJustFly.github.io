# Ceph Monitor Architecture Analysis
## Monitor Overall Architecture Overview
### Core Functional Positioning
Ceph Monitor serves as the control plane of the cluster, primarily responsible for the following core duties:
- **Cluster Map Maintenance**: Managing key mapping information including MonitorMap, OSDMap, CRUSHMap, MDSMap, PGMap, etc.
- **Status Monitoring & Health Checks**: Real-time monitoring of cluster status and generating health reports
- **Distributed Consistency Guarantee**: Ensuring cluster metadata consistency across all nodes based on Paxos algorithm
- **Authentication & Authorization**: Managing CephX authentication system and user permissions
- **Election & Arbitration**: Maintaining Monitor quorum and handling failure recovery
### Monitor Architecture Diagram
```mermaid
graph TB
    subgraph "Ceph Monitor Core Architecture"
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
    subgraph "External Interactions"
        H[OSD Daemons] --> A
        I[MDS Daemons] --> A
        J[Client Applications] --> A
        K[Admin Tools] --> A
        L[Dashboard/Grafana] --> A
    end
```
---
## Monitor Core Submodule Analysis
### MonitorStore Storage Engine
**Functional Overview**:
MonitorStore is the persistent storage engine of Monitor, implemented based on RocksDB, responsible for storing all critical cluster metadata.
**Core Architecture**:
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
### Paxos Consensus Engine
**Functional Overview**:
The Paxos engine is the most core module of Monitor, implementing distributed consensus algorithms to ensure cluster metadata consistency across all Monitor nodes.
**Architecture Design**:
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
**Core Operating Mechanism**:
1. **Proposal Phase**:
```bash
# Check current quorum status
ceph quorum_status
```
1. **Consistency Guarantee Process**:
   - **Phase 1 (Prepare Phase)**: Leader sends Prepare requests to all Acceptors
   - **Phase 2 (Accept Phase)**: Collects majority responses and sends Accept requests
   - **Commit Phase**: After reaching consensus, broadcasts commit messages to all nodes
### Election & Arbitration Module
**Functional Overview**:
Responsible for Monitor cluster Leader election, quorum maintenance, and failure detection, ensuring cluster availability under various network conditions.
**Election Strategy Architecture**:
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
**Key Configuration Parameters**:
```ini
# Monitor election related configuration
[mon]
mon_election_timeout = 5
mon_lease = 5
mon_lease_renew_interval_factor = 0.6
mon_lease_ack_timeout_factor = 2.0
mon_accept_timeout_factor = 2.0
```
**Election Trigger Conditions**:
- Monitor node startup or restart
- Network partition or connection disconnection
- Leader node failure or unresponsiveness
- Manual election trigger (operational intervention)
### Health Monitoring Module
**Functional Overview**:
Real-time monitoring of cluster component health status, generating alert information, and providing detailed diagnostic data.
**Monitoring Architecture**:
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
**Health Check Command Examples**:
```bash
# Get overall cluster status
ceph status
ceph -s
# View detailed health information
ceph health detail
# Monitor cluster status changes
ceph -w
# View specific component status
ceph pg stat
ceph osd stat
ceph mon stat
```
**Health Status Classification**:
- **HEALTH_OK**: Cluster running normally
- **HEALTH_WARN**: Warnings exist but don't affect data safety
- **HEALTH_ERR**: Errors exist requiring immediate attention
### Configuration Management Module
**Functional Overview**:
Managing cluster and daemon configuration parameters, supporting runtime configuration updates and distribution.
**Configuration Management Architecture**:
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
**Configuration Operation Commands**:
```bash
# Set global configuration
ceph config set global public_network 192.168.160.0/24
# View specific daemon configuration
ceph config show osd.0
ceph config show-with-defaults osd.0
# Set configuration key-value
ceph config-key set <key> <value>
# Generate minimal configuration file
ceph config generate-minimal-conf > /etc/ceph/ceph.conf
```
## Monitor Interaction Relationships with Other Components
### Monitor-OSD Interaction
**Interaction Mode**:
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
**Key Interaction Content**:
- **Heartbeat Detection**: OSDs periodically report their alive status to Monitor
- **Status Updates**: PG status, capacity information, performance metrics
- **Map Distribution**: OSD Map, CRUSH Map update notifications
- **Failure Handling**: OSD failure detection and marking
### Monitor-MDS Interaction
**Interaction Architecture**:
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
---
