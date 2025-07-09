# CephFS Caps 机制深度技术分析

## 🏗️ 核心架构概览

CephFS 的 capability (caps) 机制是一个复杂的分布式一致性系统，用于管理客户端对文件系统对象的访问权限。它结合了分布式锁、缓存一致性和访问控制。

### 架构组件关系图

```mermaid
graph TB
    subgraph "客户端层 (Client Layer)"
        C1[ceph-fuse Client]
        C2[kclient Client] 
        C3[libcephfs Client]
        CC[Client Cache]
        CI[Client Inode]
    end
    
    subgraph "MDS 集群 (MDS Cluster)"
        MDS1[Active MDS<br/>Primary]
        MDS2[Standby MDS]
        MDS3[Standby-Replay MDS]
    end
    
    subgraph "Capability 管理层"
        CM[Cap Manager<br/>MDSCap类]
        CL[Lock Manager<br/>SimpleLock]
        CR[Cap Revocation<br/>Locker类]
        CE[Cap Export/Import<br/>Migrator]
    end
    
    subgraph "分布式锁类型"
        AL[AUTH Lock<br/>文件属性锁]
        LL[LINK Lock<br/>目录链接锁]
        XL[XATTR Lock<br/>扩展属性锁]
        FL[FILE Lock<br/>文件数据锁]
        IL[INODE Lock<br/>inode锁]
        DL[DENTRY Lock<br/>目录项锁]
    end
    
    subgraph "存储后端"
        POOL[Metadata Pool<br/>元数据存储]
        DPOOL[Data Pool<br/>数据存储]
        JPOOL[Journal Pool<br/>日志存储]
        MON[Monitor Cluster<br/>集群状态]
    end
    
    C1 <--> |Cap Request/Grant| MDS1
    C2 <--> |Cap Request/Grant| MDS1  
    C3 <--> |Cap Request/Grant| MDS1
    
    CC --> CI
    CI --> C1
    
    MDS1 --> CM
    CM --> CL
    CM --> CR
    CM --> CE
    
    CL --> AL
    CL --> LL
    CL --> XL
    CL --> FL
    CL --> IL
    CL --> DL
    
    MDS1 <--> POOL
    MDS1 <--> JPOOL
    MDS1 <--> MON
    C1 <--> DPOOL
    C2 <--> DPOOL
    C3 <--> DPOOL
```

## 🔐 Capability 权限类型详解

### 权限位掩码定义

```cpp
// src/include/ceph_fs.h - Capability 位定义
#define CEPH_CAP_GSHARED     1   /* 共享读权限 */
#define CEPH_CAP_GEXCL       2   /* 独占写权限 */
#define CEPH_CAP_GCACHE      4   /* 缓存权限 */
#define CEPH_CAP_GRD         8   /* 读数据权限 */
#define CEPH_CAP_GWR        16   /* 写数据权限 */
#define CEPH_CAP_GBUFFER    32   /* 缓冲写权限 */
#define CEPH_CAP_GWREXTEND  64   /* 扩展写权限 */
#define CEPH_CAP_GLAZYIO   128   /* 延迟IO权限 */

// 组合权限定义
#define CEPH_CAP_AUTH_SHARED  (CEPH_CAP_GSHARED)
#define CEPH_CAP_AUTH_EXCL    (CEPH_CAP_GEXCL | CEPH_CAP_GSHARED)

#define CEPH_CAP_LINK_SHARED  (CEPH_CAP_GSHARED) 
#define CEPH_CAP_LINK_EXCL    (CEPH_CAP_GEXCL | CEPH_CAP_GSHARED)

#define CEPH_CAP_XATTR_SHARED (CEPH_CAP_GSHARED)
#define CEPH_CAP_XATTR_EXCL   (CEPH_CAP_GEXCL | CEPH_CAP_GSHARED)

#define CEPH_CAP_FILE_RD      (CEPH_CAP_GSHARED | CEPH_CAP_GRD)
#define CEPH_CAP_FILE_WR      (CEPH_CAP_GEXCL | CEPH_CAP_GWR | CEPH_CAP_GSHARED)
#define CEPH_CAP_FILE_CACHE   (CEPH_CAP_GCACHE)
#define CEPH_CAP_FILE_BUFFER  (CEPH_CAP_GBUFFER)
#define CEPH_CAP_FILE_EXCL    (CEPH_CAP_GEXCL)
#define CEPH_CAP_FILE_WR_EXTEND (CEPH_CAP_GWREXTEND)
#define CEPH_CAP_FILE_LAZYIO  (CEPH_CAP_GLAZYIO)
```

### Capability 类型映射图

```mermaid
graph LR
    subgraph "AUTH Cap - 文件属性"
        A1[AUTH_SHARED<br/>读属性]
        A2[AUTH_EXCL<br/>写属性]
    end
    
    subgraph "LINK Cap - 目录链接"
        L1[LINK_SHARED<br/>读链接]
        L2[LINK_EXCL<br/>写链接]
    end
    
    subgraph "XATTR Cap - 扩展属性"
        X1[XATTR_SHARED<br/>读扩展属性]
        X2[XATTR_EXCL<br/>写扩展属性]
    end
    
    subgraph "FILE Cap - 文件数据"
        F1[FILE_RD<br/>读数据]
        F2[FILE_WR<br/>写数据]
        F3[FILE_CACHE<br/>缓存数据]
        F4[FILE_BUFFER<br/>缓冲写入]
        F5[FILE_EXCL<br/>独占访问]
        F6[FILE_LAZYIO<br/>延迟IO]
    end
    
    A1 --> |stat,getattr| FileOps
    A2 --> |chmod,chown| FileOps
    L1 --> |readdir| DirOps
    L2 --> |mkdir,rmdir| DirOps
    X1 --> |getxattr| XattrOps
    X2 --> |setxattr| XattrOps
    F1 --> |read| IOOps
    F2 --> |write| IOOps
    F3 --> |page cache| CacheOps
    F4 --> |write buffer| CacheOps
```

## 🔄 Lock 状态机和转换

### SimpleLock 状态机

```mermaid
stateDiagram-v2
    [*] --> LOCK_SYNC
    
    LOCK_SYNC --> LOCK_LOCK: need_write
    LOCK_SYNC --> LOCK_MIX: need_mixed
    LOCK_SYNC --> LOCK_TSYN: need_tsyn
    
    LOCK_LOCK --> LOCK_SYNC: no_writers
    LOCK_LOCK --> LOCK_XLOCK: need_xlock
    LOCK_LOCK --> LOCK_XLOCKDONE: xlock_finish
    
    LOCK_MIX --> LOCK_SYNC: no_readers_writers
    LOCK_MIX --> LOCK_LOCK: need_write_only
    
    LOCK_XLOCK --> LOCK_XLOCKDONE: xlock_finish
    LOCK_XLOCKDONE --> LOCK_LOCK: xlock_done
    LOCK_XLOCKDONE --> LOCK_SYNC: release_all
    
    LOCK_TSYN --> LOCK_SYNC: tsyn_finish
    
    state LOCK_SYNC {
        [*] --> Stable
        Stable --> MultipleReaders
        MultipleReaders --> Stable
    }
    
    state LOCK_LOCK {
        [*] --> SingleWriter  
        SingleWriter --> ExclusiveWrite
    }
    
    state LOCK_MIX {
        [*] --> MixedAccess
        MixedAccess --> ReadersAndWriters
    }
```

## 🔄 Cap 工作流程详解

### 完整的 Capability 生命周期

```mermaid
sequenceDiagram
    participant C as Client
    participant MDS as Active MDS
    participant L as Locker
    participant CM as CapManager
    participant SL as SimpleLock
    participant J as Journal
    
    Note over C,J: Phase 1: Cap 请求
    C->>MDS: MClientRequest(open)
    MDS->>L: acquire_locks(inode)
    L->>SL: lock(FILE_LOCK, LOCK_LOCK)
    
    alt Lock Available
        SL-->>L: lock_granted
        L->>CM: issue_caps(client, inode)
        CM->>CM: create_capability()
        Note over CM: 创建 Capability 对象
        CM->>J: journal_cap_grant()
        J-->>CM: journal_committed
        CM->>C: MClientCaps(grant)
    else Lock Contention
        SL->>SL: add_waiter(client)
        Note over SL: 排队等待锁释放
        SL-->>L: lock_available_callback
        L->>CM: issue_caps()
        CM->>C: MClientCaps(grant)
    end
    
    Note over C,J: Phase 2: Cap 使用
    C->>C: install_caps()
    C->>C: perform_file_io()
    
    Note over C,J: Phase 3: Cap 撤销
    MDS->>CM: revoke_caps(mask)
    CM->>C: MClientCaps(revoke)
    C->>C: flush_dirty_caps()
    C->>C: invalidate_cache()
    C->>MDS: MClientCaps(release)
    MDS->>CM: process_cap_release()
    CM->>SL: unlock()
    SL->>SL: notify_waiters()
```

## 🏛️ 核心数据结构

### 客户端 Capability 结构

```cpp
// src/client/Inode.h
class Cap {
public:
    MetaSession *session;        // MDS 会话指针
    uint64_t cap_id;            // Capability ID
    unsigned issued;            // 已发放的权限位
    unsigned implemented;       // 已实现的权限位
    unsigned wanted;            // 想要的权限位
    unsigned pending;           // 待处理的权限位
    
    utime_t last_used;          // 最后使用时间
    int64_t gen;               // 生成版本号
    int64_t cap_gen;           // Cap 生成号
    int64_t seq;               // 序列号
    int64_t issue_seq;         // 发放序列号
    int64_t mseq;              // MDS 序列号
    
    // Cap 权限检查
    bool is_valid() const { return session != nullptr; }
    bool issued_caps_need_check() const;
    void touch() { last_used = ceph_clock_now(); }
};

// 客户端 Inode 扩展
class Inode {
    // ... 其他成员
    std::map<mds_rank_t, Cap> caps;  // 各 MDS 的 caps
    unsigned caps_issued() const;    // 已发放的所有 caps
    unsigned caps_wanted() const;    // 想要的所有 caps
    void get_caps_issued(unsigned *issued, unsigned *implemented);
};
```

### MDS 端 Capability 结构

```cpp
// src/mds/Capability.h
class Capability {
    client_t client;            // 客户端标识
    CInode *inode;             // 指向的 inode
    uint64_t cap_id;           // Cap ID
    
    unsigned issued_;          // 已发放权限
    unsigned pending_;         // 待处理权限  
    unsigned wanted_;          // 客户端想要的权限
    
    utime_t last_sent;         // 最后发送时间
    utime_t last_revoke_stamp; // 最后撤销时间
    int64_t trans_seq;         // 事务序列号
    int64_t client_follows;    // 客户端跟随序列号
    
public:
    // 权限管理方法
    void set_wanted(unsigned w) { wanted_ = w; }
    void inc_suppress() { suppress++; }
    void dec_suppress() { suppress--; }
    
    bool is_suppress() const { return suppress > 0; }
    bool is_stale() const;
    bool is_valid() const { return client > 0; }
    
    // 权限检查
    unsigned issued() const { return issued_; }
    unsigned pending() const { return pending_; }
    unsigned wanted() const { return wanted_; }
};
```

## 🔧 关键函数实现

### Cap 发放核心函数

```cpp
// src/mds/Locker.cc
void Locker::issue_caps(CInode *in, Capability *cap) {
    dout(7) << "issue_caps for " << *in << " to client." << cap->get_client() << dendl;
    
    unsigned was_issued = cap->issued();
    unsigned wanted = cap->wanted();
    unsigned issued = 0;
    
    // 检查各种锁的状态来决定可以发放的权限
    
    // AUTH cap - 文件属性权限
    if (in->authlock.can_read(cap->get_client())) {
        issued |= CEPH_CAP_AUTH_SHARED;
    }
    if (in->authlock.can_write(cap->get_client())) {
        issued |= CEPH_CAP_AUTH_EXCL;
    }
    
    // LINK cap - 目录链接权限
    if (in->linklock.can_read(cap->get_client())) {
        issued |= CEPH_CAP_LINK_SHARED;
    }
    if (in->linklock.can_write(cap->get_client())) {
        issued |= CEPH_CAP_LINK_EXCL;
    }
    
    // XATTR cap - 扩展属性权限
    if (in->xattrlock.can_read(cap->get_client())) {
        issued |= CEPH_CAP_XATTR_SHARED;
    }
    if (in->xattrlock.can_write(cap->get_client())) {
        issued |= CEPH_CAP_XATTR_EXCL;
    }
    
    // FILE cap - 文件数据权限 (最复杂)
    if (in->filelock.can_read(cap->get_client())) {
        issued |= CEPH_CAP_FILE_RD;
        if (in->filelock.can_read_projected(cap->get_client())) {
            issued |= CEPH_CAP_FILE_CACHE;
        }
    }
    
    if (in->filelock.can_write(cap->get_client())) {
        issued |= CEPH_CAP_FILE_WR;
        if (in->filelock.can_write_projected(cap->get_client())) {
            issued |= CEPH_CAP_FILE_BUFFER;
            if (in->filelock.get_state() == LOCK_EXCL) {
                issued |= CEPH_CAP_FILE_EXCL;
            }
        }
    }
    
    // 限制权限为客户端实际想要的
    issued &= wanted;
    
    // 如果权限有变化，发送 grant 消息
    if (issued != was_issued) {
        cap->set_issued(issued);
        send_cap_grant(cap, issued);
        
        // 记录到日志
        if (mds->mdlog->get_write_pos() > 0) {
            mds->mdlog->submit_entry(new EMetaBlob(mds->mdlog));
        }
    }
}

// Cap 撤销核心函数
void Locker::revoke_caps(CInode *in, int revoke_mask, client_t client) {
    dout(7) << "revoke_caps " << ccap_string(revoke_mask) 
            << " on " << *in << dendl;
    
    auto it = in->get_client_caps().find(client);
    if (it == in->get_client_caps().end()) {
        return; // 客户端没有 caps
    }
    
    Capability *cap = it->second;
    unsigned revoking = cap->issued() & revoke_mask;
    
    if (revoking) {
        dout(7) << " revoking " << ccap_string(revoking) 
                << " from client." << client << dendl;
        
        cap->set_pending(cap->pending() | revoking);
        cap->set_issued(cap->issued() & ~revoking);
        
        // 发送撤销消息
        send_cap_revoke(cap, revoking);
        
        // 设置撤销超时
        if (!cap->is_suppress()) {
            mds->locker->set_cap_revoke_timeout(cap);
        }
    }
}
```

### 锁状态检查函数

```cpp
// src/mds/locks.cc
bool SimpleLock::can_read(client_t client) {
    switch (state) {
        case LOCK_SYNC:
            return true;  // 同步状态允许所有客户端读
            
        case LOCK_MIX:
            return true;  // 混合状态允许读
            
        case LOCK_LOCK:
            // 锁定状态只允许锁持有者读
            return is_rdlocked_by(client) || is_wrlocked_by(client);
            
        case LOCK_XLOCK:
            // 排他锁状态只允许锁持有者
            return is_xlocked_by(client);
            
        default:
            return false;
    }
}

bool SimpleLock::can_write(client_t client) {
    switch (state) {
        case LOCK_LOCK:
            return is_wrlocked_by(client);
            
        case LOCK_XLOCK:
            return is_xlocked_by(client);
            
        default:
            return false;
    }
}

// 锁状态转换
void SimpleLock::go_lock() {
    dout(7) << "go_lock on " << *get_parent() << dendl;
    
    state = LOCK_LOCK;
    
    // 撤销所有客户端的读权限，除了获得写锁的客户端
    for (auto& p : parent->get_client_caps()) {
        client_t client = p.first;
        if (client != lock_client) {
            revoke_client_caps(client, CEPH_CAP_FILE_RD);
        }
    }
}
```

## 🔄 分布式一致性保证机制

### 一致性层次模型

```mermaid
graph TD
    subgraph "一致性级别"
        L1[Client Cache<br/>最终一致性<br/>Client-side caching]
        L2[MDS Memory<br/>顺序一致性<br/>In-memory metadata]
        L3[Journal WAL<br/>强一致性<br/>Write-ahead logging]  
        L4[RADOS Storage<br/>线性一致性<br/>Distributed storage]
    end
    
    subgraph "一致性机制"
        M1[Cap Revocation<br/>缓存失效协议]
        M2[Lock Ordering<br/>死锁避免]
        M3[Journal Commit<br/>事务保证]
        M4[RADOS ACID<br/>原子操作]
    end
    
    subgraph "故障处理"
        F1[Client Failure<br/>Cap 超时回收]
        F2[MDS Failure<br/>Cap 重建]
        F3[Network Partition<br/>脑裂处理]
        F4[Storage Failure<br/>副本恢复]
    end
    
    L1 --> |Cache Coherence| M1
    L2 --> |Mutual Exclusion| M2  
    L3 --> |Durability| M3
    L4 --> |Atomicity| M4
    
    M1 --> F1
    M2 --> F2
    M3 --> F3
    M4 --> F4
    
    F1 --> L2
    F2 --> L3
    F3 --> L4
    F4 --> L1
```

### Cap 迁移流程 (MDS Failover)

```mermaid
sequenceDiagram
    participant C as Client
    participant MDS1 as Failed MDS
    participant MDS2 as Takeover MDS
    participant MON as Monitor
    participant RADOS as RADOS Store
    
    Note over C,RADOS: MDS 故障检测
    MDS1->>X: 故障/网络分区
    MON->>MON: 检测 MDS1 故障
    MON->>MDS2: assign_rank(failed_rank)
    
    Note over C,RADOS: Cap 状态重建
    MDS2->>RADOS: 读取 journal 和 metadata
    RADOS-->>MDS2: 返回持久化状态
    MDS2->>MDS2: replay_journal()
    MDS2->>MDS2: rebuild_cap_state()
    
    Note over C,RADOS: 客户端重连
    C->>MDS2: 重新连接请求
    MDS2->>C: MClientSession(renewal)
    C->>MDS2: 报告当前 caps 状态
    
    Note over C,RADOS: Cap 状态同步
    MDS2->>MDS2: validate_client_caps()
    alt Caps Valid
        MDS2->>C: MClientCaps(confirm)
    else Caps Invalid  
        MDS2->>C: MClientCaps(revoke_all)
        C->>C: flush_and_invalidate()
        C->>MDS2: 重新请求所需 caps
        MDS2->>C: MClientCaps(grant)
    end
```

## 📊 性能优化策略

### Cap 缓存优化

```cpp
// src/client/Client.cc - 客户端 Cap 缓存优化
class Client {
    // Cap 缓存管理
    LRUObjects cap_lru;        // Cap LRU 缓存
    uint64_t max_caps_cache;   // 最大缓存 caps 数量
    
    void trim_caps() {
        while (cap_lru.lru_get_size() > max_caps_cache) {
            Inode *in = static_cast<Inode*>(cap_lru.lru_expire());
            if (in) {
                release_caps(in, CEPH_CAP_FILE_CACHE);
            }
        }
    }
    
    // 智能 Cap 预测
    void predict_caps_needed(Inode *in, unsigned &wanted) {
        // 基于访问模式预测需要的权限
        if (in->access_pattern & ACCESS_PATTERN_SEQUENTIAL) {
            wanted |= CEPH_CAP_FILE_CACHE;
        }
        if (in->access_pattern & ACCESS_PATTERN_RANDOM) {
            wanted |= CEPH_CAP_FILE_RD;
        }
        if (in->dirty_pages > 0) {
            wanted |= CEPH_CAP_FILE_BUFFER;
        }
    }
};
```

