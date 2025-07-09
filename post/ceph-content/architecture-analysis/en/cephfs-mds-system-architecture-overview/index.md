## MDS System Architecture Overview

Ceph MDS is the core component of CephFS (Ceph File System), responsible for handling all file system metadata operations. MDS adopts a distributed, scalable architecture that supports multi-active MDS and dynamic load balancing.

### MDS Position in Ceph Ecosystem

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
### MDS Module Architecture

![image.png|600](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images20250530152434.png)

## MDS Core Submodule Architecture Analysis

### MDSMap Management Module

MDSMap is the core module for MDS cluster state management, responsible for maintaining MDS cluster topology information and status.

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

**Core Functions:**
- MDS cluster member management
- Rank allocation and failover
- Filesystem to MDS mapping
- State synchronization and version control

**Key Configuration Parameters:**
```bash
# Set maximum active MDS count
ceph fs set <fsname> max_mds <count>

# View MDS status
ceph mds stat
ceph fs status <fsname>
```

### Session Management Module

The Session management module handles client connections and session state maintenance.

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

**Core Functions:**
- Client connection authentication
- Session lifecycle management
- Capability distribution and revocation
- Session timeout handling

### MDCache Module

MDCache is the core caching module of MDS, responsible for metadata caching, consistency maintenance, and distributed coordination.

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

**Core Functions:**
- Distributed metadata caching
- Cache consistency protocol
- Metadata prefetching and predictive caching
- Memory management and LRU eviction

### MDS Balancer Module

The load balancing module ensures metadata load is evenly distributed among multiple active MDS.

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

**Core Configuration:**
```bash
# Enable MDS load balancing
ceph config set mds mds_bal_mode 2

# Set load balancing thresholds
ceph config set mds mds_bal_need_min 0.2
ceph config set mds mds_bal_need_max 1.25
```

### Journal and Recovery Module

The MDS Journal module is responsible for persistent logging of metadata operations and failure recovery.

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

## MDS State Machine and Lifecycle

### MDS State Transition Diagram

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
        Normal working state
        Processing client requests
    end note
    
    note right of standby
        Standby state
        Waiting for rank assignment
    end note
    
    note right of resolve
        Resolving metadata conflicts
        Handling distributed state
    end note
```

### MDS Startup and Initialization Flow

```mermaid
sequenceDiagram
    participant Monitor as Monitor
    participant MDS as MDS Daemon  
    participant RADOS as RADOS
    participant Standby as Standby MDS
    
    MDS->>Monitor: Register as standby
    Monitor->>MDS: Assign rank
    MDS->>RADOS: Read journal
    MDS->>MDS: Replay journal
    MDS->>Monitor: Report active status
    Monitor->>Standby: Notify status change
    MDS->>MDS: Start processing client requests
```

## MDS Relationships with Upstream and Downstream Components

### MDS-Monitor Interaction

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

**Key Monitoring Commands:**
```bash
# View MDS health status
ceph health detail

# Check slow metadata IO
ceph mds perf dump

# View MDS alerts
ceph health mute MDS_SLOW_REQUEST

# MDS performance statistics
ceph daemon mds.<name> perf dump
```

### MDS-RADOS Interaction

MDS stores all persistent metadata through RADOS:

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

### MDS-Client Interaction

```mermaid
sequenceDiagram
    participant Client as CephFS Client
    participant MDS as MDS Active
    participant RADOS as RADOS
    
    Client->>MDS: Establish session
    MDS->>Client: Distribute capabilities
    Client->>MDS: Metadata request (open/mkdir/stat)
    MDS->>RADOS: Read/update metadata
    RADOS->>MDS: Return results
    MDS->>Client: Return metadata
    Client->>MDS: Release capabilities
    MDS->>Client: Confirm release
```


## Common MDS Troubleshooting and Recovery


```bash
# Check MDS status
ceph fs status
ceph mds stat


# MDS performance analysis
ceph daemon mds.<name> perf dump
ceph daemon mds.<name> dump cache

# Check client connections
ceph daemon mds.<name> session ls
```



## MDS Monitoring and Alerting

### Key Performance Indicators

```bash
# MDS performance monitoring
ceph daemonperf mds

# Key metrics:
# - mds.inodes: Number of cached inodes
# - mds.reply_latency: Response latency
# - mds.request_rate: Request rate
# - mds.sessions: Active session count
```

### Alert Configuration

Important alert items:
- MDS_SLOW_REQUEST: Slow request alert
- MDS_SLOW_METADATA_IO: Slow metadata IO
- MDS_INSUFFICIENT_STANDBY: Insufficient standby MDS
- MDS_HEALTH_READ_ONLY: MDS read-only status
