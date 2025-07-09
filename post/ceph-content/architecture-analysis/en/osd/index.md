### Classical OSD Overall Architecture Diagram

```mermaid
graph TB
    subgraph "OSD Process Architecture"
        OSD[OSD Main Class]
        OSDService[OSDService<br/>Core Service]
        ShardedOpWQ[ShardedOpWQ<br/>Sharded Operation Queue]
        Messenger[Messenger<br/>Message System]
    end
    
    subgraph "PG Management Subsystem"
        PGMap[pg_map<br/>PG Mapping Table]
        PG[PG Class<br/>Placement Group]
        PGBackend[PGBackend<br/>Backend Implementation]
        ReplicatedBackend[ReplicatedBackend]
        ECBackend[ECBackend]
    end
    
    subgraph "Object Storage Subsystem"
        ObjectStore[ObjectStore<br/>Storage Abstraction Layer]
        FileStore[FileStore<br/>Filesystem Storage]
        BlueStore[BlueStore<br/>Raw Device Storage]
        ObjectContext[ObjectContext<br/>Object Context]
    end
    
    subgraph "Recovery Subsystem"
        RecoveryState[RecoveryState<br/>Recovery State Machine]
        PeeringState[PeeringState<br/>Peering State]
        BackfillState[BackfillState<br/>Backfill State]
        RecoveryWQ[RecoveryWQ<br/>Recovery Work Queue]
    end
    
    subgraph "Monitoring & Statistics"
        PGStats[PGStats<br/>PG Statistics]
        OSDStats[OSDStats<br/>OSD Statistics]
        PerfCounters[PerfCounters<br/>Performance Counters]
        Logger[Logger<br/>Logging System]
    end
    
    OSD --> OSDService
    OSD --> ShardedOpWQ
    OSD --> Messenger
    OSD --> PGMap
    PGMap --> PG
    PG --> PGBackend
    PGBackend --> ReplicatedBackend
    PGBackend --> ECBackend
    PG --> ObjectStore
    ObjectStore --> FileStore
    ObjectStore --> BlueStore
    PG --> ObjectContext
    PG --> RecoveryState
    RecoveryState --> PeeringState
    RecoveryState --> BackfillState
    OSD --> RecoveryWQ
    PG --> PGStats
    OSD --> OSDStats
    OSD --> PerfCounters
    OSD --> Logger
```

### OSD Core Class Structure Details

```mermaid
classDiagram
    class OSD {
        -int whoami
        -Messenger* cluster_messenger
        -Messenger* client_messenger
        -MonClient* monc
        -MgrClient* mgrc
        -ObjectStore* store
        -OSDService service
        -map~spg_t,PG*~ pg_map
        -RWLock pg_map_lock
        -OSDMapRef osdmap
        -epoch_t up_epoch
        -ThreadPool op_tp
        -ShardedOpWQ op_sharded_wq
        -RecoveryWQ recovery_wq
        -SnapTrimWQ snap_trim_wq
        -ScrubWQ scrub_wq
        +handle_osd_op(MOSDOp* op)
        +handle_replica_op(MOSDSubOp* op)
        +handle_pg_create(MOSDPGCreate* m)
        +handle_osd_map(MOSDMap* m)
        +process_peering_events()
        +start_boot()
        +shutdown()
    }
    
    class OSDService {
        -OSD* osd
        -CephContext* cct
        -ObjectStore* store
        -LogClient* log_client
        -PGRecoveryStats recovery_stats
        -Throttle recovery_ops_throttle
        -Throttle recovery_bytes_throttle
        -ClassHandler* class_handler
        -map~hobject_t,ObjectContext*~ object_contexts
        -LRUExpireMap object_context_lru
        +get_object_context(hobject_t oid)
        +release_object_context(ObjectContext* obc)
        +queue_for_recovery(PG* pg)
        +queue_for_scrub(PG* pg)
    }
    
    class ShardedOpWQ {
        -vector~OpWQ*~ shards
        -atomic~uint32_t~ next_shard
        +queue(OpRequestRef op)
        +dequeue(OpWQ* shard)
        +process_batch()
    }
    
    class OpWQ {
        -ThreadPool::TPHandle* handle
        -list~OpRequestRef~ ops
        -Mutex ops_lock
        +enqueue_front(OpRequestRef op)
        +enqueue_back(OpRequestRef op)
        +dequeue()
        +process()
    }
    
    OSD --> OSDService
    OSD --> ShardedOpWQ
    ShardedOpWQ --> OpWQ
```

### PG Class Detailed Structure

```mermaid
classDiagram
    class PG {
        -spg_t pg_id
        -OSDService* osd
        -CephContext* cct
        -PGBackend* pgbackend
        -ObjectStore::CollectionHandle ch
        -RecoveryState recovery_state
        -PGLog pg_log
        -IndexedLog projected_log
        -eversion_t last_update
        -epoch_t last_epoch_started
        -set~pg_shard_t~ up
        -set~pg_shard_t~ acting
        -map~hobject_t,ObjectContext*~ object_contexts
        -Mutex pg_lock
        -Cond pg_cond
        -list~OpRequestRef~ waiting_for_peered
        -list~OpRequestRef~ waiting_for_active
        -map~eversion_t,list~OpRequestRef~~ waiting_for_ondisk
        +do_request(OpRequestRef op)
        +do_op(OpRequestRef op)
        +do_sub_op(OpRequestRef op)
        +execute_ctx(OpContext* ctx)
        +issue_repop(RepGather* repop)
        +eval_repop(RepGather* repop)
        +start_recovery_ops()
        +recover_object()
        +on_change(ObjectStore::Transaction* t)
        +activate()
        +clean_up_local()
    }
    
    class RecoveryState {
        -PG* pg
        -RecoveryMachine machine
        -boost::statechart::state_machine base
        +handle_event(const boost::statechart::event_base& evt)
        +process_peering_events()
        +advance_map()
        +need_up_thru()
    }
    
    class PGLog {
        -IndexedLog log
        -eversion_t tail
        -eversion_t head
        -list~pg_log_entry_t~ pending_log
        -set~eversion_t~ pending_dups
        +add(pg_log_entry_t& entry)
        +trim(eversion_t trim_to)
        +merge_log(ObjectStore::Transaction* t)
        +write_log_and_missing()
    }
    
    class PGBackend {
        -PG* parent
        -ObjectStore* store
        -CephContext* cct
        +submit_transaction()
        +objects_list_partial()
        +objects_list_range()
        +objects_get_attr()
        +objects_read_sync()
        +be_deep_scrub()
    }
    
    PG --> RecoveryState
    PG --> PGLog
    PG --> PGBackend
```

### Read/Write IO Processing Detailed Flow

#### Write Operation Complete Flow

```mermaid
sequenceDiagram
    participant Client
    participant OSD
    participant PG
    participant OpWQ
    participant ObjectStore
    participant Journal
    participant Replica
    
    Client->>OSD: MOSDOp(write)
    OSD->>OSD: handle_osd_op()
    Note right of OSD: 1. Validate request<br/>2. Build OpRequest<br/>3. Find PG
    
    OSD->>OpWQ: enqueue(OpRequest)
    OpWQ->>PG: do_request()
    
    PG->>PG: Check PG state
    alt PG Active
        PG->>PG: do_op()
        PG->>PG: execute_ctx()
        Note right of PG: Build OpContext
        
        PG->>ObjectStore: queue_transaction()
        Note right of ObjectStore: Write to local storage
        
        alt Is Primary and has replicas
            PG->>Replica: Send SubOp
            Note right of PG: Send to all replicas
            
            Replica->>ObjectStore: queue_transaction()
            Replica->>PG: SubOpReply
            
            PG->>PG: eval_repop()
            Note right of PG: Wait for all replica confirmations
        end
        
        ObjectStore->>Journal: write_journal()
        Journal->>PG: on_applied()
        PG->>Client: MOSDOpReply
        
        Journal->>ObjectStore: apply_transaction()
        ObjectStore->>PG: on_commit()
        
    else PG not Active
        PG->>PG: Add to waiting_for_active queue
    end
```

#### Read Operation Flow

```mermaid
sequenceDiagram
    participant Client
    participant OSD
    participant PG
    participant ObjectContext
    participant ObjectStore
    
    Client->>OSD: MOSDOp(read)
    OSD->>PG: do_request()
    
    PG->>PG: Check PG state
    alt PG readable
        PG->>ObjectContext: get_object_context()
        ObjectContext->>ObjectStore: getattr()
        ObjectStore-->>ObjectContext: Object metadata
        
        PG->>ObjectStore: read()
        ObjectStore-->>PG: Object data
        
        PG->>Client: MOSDOpReply(data)
    else PG not readable
        PG->>PG: Add to waiting queue
    end
```
