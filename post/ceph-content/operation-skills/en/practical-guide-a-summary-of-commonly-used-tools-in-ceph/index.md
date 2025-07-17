# Practical Guide: Ceph Command Tools Summary
## 📋 Common Tools (Summary Overview)
| Function Category | Main Commands | Verification Status | Usage Frequency | Application Scenarios | Risk Level |
|---------|---------|---------|---------|---------|---------|
| **Cluster Monitoring** | `ceph -s`, `ceph health(detail)`, `ceph df`, `ceph -w`| ✅ Verified | ⭐⭐⭐⭐⭐ | Daily monitoring, troubleshooting | 🟢 No risk |
| **I/O Monitoring** | `ceph iostat`(version dependent,N+), `ceph -w`, `ceph status` | ✅ Verified | ⭐⭐⭐⭐ | Performance monitoring | 🟢 No risk |
| **OSD Management** | `ceph osd tree`, `ceph osd status`, `ceph osd out/in` | ✅ Verified | ⭐⭐⭐⭐ | OSD maintenance, capacity management | 🟡 Medium risk (queries safe) |
| **Monitor Management** | `ceph mon stat`, `ceph quorum_status`, `ceph mon add/remove` | ✅ Verified | ⭐⭐⭐ | Cluster management, high availability | 🔴 High risk (queries safe)|
| **Manager Management** | `ceph mgr module enable/disable`, `ceph mgr stat` | ✅ Verified | ⭐⭐⭐ | Feature management, dashboard | 🟡 Medium risk (queries safe) |
| **Pool Management** | `ceph osd pool create/delete`, `ceph osd pool set` | ✅ Verified | ⭐⭐⭐⭐ | Storage planning, quota management | 🔴 High risk (queries safe) |
| **PG Management** | `ceph pg stat`, `ceph pg repair`, `ceph pg scrub` | ✅ Verified | ⭐⭐⭐⭐ | Data integrity, fault repair | 🟡 Medium risk (queries safe) |
| **Authentication Management** | `ceph auth list/create/del` | ✅ Verified | ⭐⭐⭐ | Security management, access control | 🔴 High risk (queries safe) |
| **CRUSH Management** | `ceph osd crush tree`, `crushtool`, `ceph osd crush rule` | ✅ Verified | ⭐⭐ | Data distribution, failure domains | 🔴 High risk (queries safe) |
| **RBD Management** | `rbd create/rm`, `rbd snap create`, `rbd map/unmap` | ✅ Verified | ⭐⭐⭐⭐ | Block storage, snapshot management | 🟡 Medium risk |
| **CephFS Management** | `ceph fs status`, `ceph mds stat`, `ceph fs dump`, `ceph mds fail` | ✅ Verified | ⭐⭐⭐ | File system, metadata | 🟡 Medium risk (queries safe) |
| **RGW Management** | `radosgw-admin user create`, `radosgw-admin bucket` | ✅ Verified | ⭐⭐⭐ | Object storage, user management | 🟡 Medium risk (queries safe)|
| **Configuration Management** | `ceph config set/get`, `ceph tell` | Not verified | ⭐⭐⭐⭐ | Parameter tuning, fault handling | 🟡 Medium risk (queries safe) |
| **Performance Analysis** | `ceph osd perf`,`rbd perf image iostat`, cephfs-top | ✅ Verified | ⭐⭐⭐ | Performance testing, bottleneck analysis | 🟢 No risk |
| **Specialized Tools** | `ceph-objectstore-tool`, `ceph-bluestore-tool` | ✅ Verified | ⭐⭐ | Data recovery, deep diagnostics | 🔴 High risk (queries safe) |
| **Troubleshooting** | `journalctl`, `ceph daemon dump`, log analysis | ✅ Verified | ⭐⭐⭐⭐ | Problem diagnosis, root cause analysis | 🟢 No risk |
| **Backup Recovery** | `ceph mon getmap`, `ceph auth export`, data export | ✅ Verified | ⭐⭐ | Disaster recovery, migration | 🟡 Medium risk (queries safe) |
## 🔧 1. Cluster Status Monitoring
### 1.1 Overall Cluster Status
```bash
# View cluster status (most commonly used)
ceph status
ceph -s
# View cluster health status
ceph health
ceph health detail
# Real-time cluster monitoring
ceph -w
# View cluster storage usage
ceph df
ceph df detail
# View cluster version information
ceph version
ceph versions
```
### 1.2 Cluster Performance Monitoring
```bash
# View cluster I/O statistics
ceph iostat
watch "ceph iostat"
# View OSD performance statistics
ceph osd perf
# View slow requests
ceph daemon osd.X dump_slow_requests
ceph daemon osd.X dump_historic_slow_ops
# PG distribution analysis
ceph pg dump | grep ^pg | awk '{print $1,$15}' | sort
```
---
## 🗄️ 2. OSD Management
### 2.1 Basic OSD Operations
```bash
# View OSD status
ceph osd stat
ceph osd dump
ceph osd tree
# View OSD usage
ceph osd df
ceph osd df tree
# Start/stop OSD
systemctl start ceph-osd@X
systemctl stop ceph-osd@X
systemctl restart ceph-osd@X
# Mark OSD as out/in
ceph osd out X
ceph osd in X
# Mark OSD as down/up
ceph osd down X
ceph osd up X
```
### 2.2 OSD Maintenance Operations
```bash
# Safely remove OSD (complete workflow)
ceph osd out X                              # Mark as out, start data migration
ceph osd safe-to-destroy X                  # Check if safe to remove
ceph osd destroy X --yes-i-really-mean-it   # Destroy OSD
ceph osd crush remove osd.X                 # Remove from CRUSH map
ceph auth del osd.X                         # Delete authentication info
ceph osd rm X                               # Remove from cluster
# Replace failed OSD
ceph osd destroy X --yes-i-really-mean-it
# Reweight OSD
ceph osd reweight X 0.8                     # Temporary weight adjustment
ceph osd crush reweight osd.X 2.0           # Permanent weight adjustment
ceph osd reweight-by-utilization            # Auto adjust weight by utilization
```
### 2.3 OSD Troubleshooting
```bash
# View detailed OSD information
ceph osd find X
ceph osd metadata X
# View OSD logs
journalctl -u ceph-osd@X -f
journalctl -u ceph-osd@X --since "1 hour ago"
# Repair OSD
ceph osd scrub X
ceph osd deep-scrub X
# View PGs on OSD
ceph pg ls-by-osd X
# OSD performance diagnostics
ceph daemon osd.X perf dump
ceph daemon osd.X dump_ops_in_flight
ceph daemon osd.X dump_blocked_ops
```
---
## 🏛️ 3. Monitor Management
### 3.1 Basic Monitor Operations
```bash
# View Monitor status
ceph mon stat
ceph mon dump
# View Monitor quorum status
ceph quorum_status
# Add Monitor
ceph mon add <mon-id> <mon-addr>
# Remove Monitor
ceph mon remove <mon-id>
# View Monitor map
ceph mon getmap -o /tmp/monmap
monmaptool --print /tmp/monmap
```
### 3.2 Monitor Maintenance
```bash
# Start/stop Monitor
systemctl start ceph-mon@<hostname>
systemctl stop ceph-mon@<hostname>
# Compact Monitor database
ceph tell mon.<mon-id> compact
# Monitor time synchronization
ceph time-sync-status
# Monitor failure recovery
ceph-mon --extract-monmap /tmp/monmap --mon-data /var/lib/ceph/mon/ceph-<id>
```
---
## 👨‍💼 4. Manager Management
### 4.1 Basic Manager Operations
```bash
# View Manager status
ceph mgr stat
ceph mgr dump
# Enable/disable Manager modules
ceph mgr module enable <module>
ceph mgr module disable <module>
# View available modules
ceph mgr module ls
# Failover Manager
ceph mgr fail <mgr-id>
```
### 4.2 Common Manager Modules
```bash
# Dashboard module
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert
ceph dashboard ac-user-create <username> -i <password-file>
ceph dashboard ac-role-add-scope-perms admin read-write pool
ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
ceph config set mgr mgr/dashboard/server_port 8443
# Prometheus module
ceph mgr module enable prometheus
ceph config set mgr mgr/prometheus/server_addr 0.0.0.0
ceph config set mgr mgr/prometheus/server_port 9283
# Balancer module
ceph mgr module enable balancer
ceph balancer mode upmap
ceph balancer on
ceph balancer status
# Alerts module
ceph mgr module enable alerts
```
---
## 🗂️ 5. Pool Management
### 5.1 Basic Pool Operations
```bash
# Create pool
ceph osd pool create <poolname> <pg_num> [pgp_num]
ceph osd pool create rbd 128 128 replicated
ceph osd pool create mypool 64 64 erasure
# Delete pool
ceph config set mon mon_allow_pool_delete true
ceph osd pool delete <poolname> <poolname> --yes-i-really-really-mean-it
# View pools
ceph osd lspools
ceph osd pool ls detail
# Set pool quota
ceph osd pool set-quota <poolname> max_objects 1000
ceph osd pool set-quota <poolname> max_bytes 1TB
# View pool statistics
ceph osd pool stats
ceph osd pool stats <poolname>
```
### 5.2 Pool Parameter Configuration
```bash
# Set pool replica count
ceph osd pool set <poolname> size 3
ceph osd pool set <poolname> min_size 2
# Set PG count
ceph osd pool set <poolname> pg_num 128
ceph osd pool set <poolname> pgp_num 128
# Set pool type
ceph osd pool set <poolname> crush_rule <rule-name>
# Enable/disable features
ceph osd pool set <poolname> noscrub true
ceph osd pool set <poolname> nodeep-scrub true
# Pool application tags
ceph osd pool application enable <poolname> rbd
ceph osd pool application enable <poolname> cephfs
ceph osd pool application enable <poolname> rgw
```
---
## 📄 6. Placement Group (PG) Management
### 6.1 PG Status Viewing
```bash
# View PG status
ceph pg stat
ceph pg dump
ceph pg dump --format json-pretty
# View abnormal PGs
ceph pg dump_stuck
ceph pg dump_stuck inactive
ceph pg dump_stuck unclean
# View specific PG information
ceph pg <pgid> query
ceph pg map <pgid>
```
### 6.2 PG Repair Operations
```bash
# Repair PG
ceph pg repair <pgid>
ceph pg scrub <pgid>
ceph pg deep-scrub <pgid>
# Force repair inconsistent objects
ceph pg force-recovery <pgid>
ceph pg force-backfill <pgid>
# View PG's OSDs
ceph pg ls-by-pool <poolname>
ceph pg ls-by-primary <osd-id>
# PG auto repair
ceph config set osd osd_scrub_auto_repair true
```
---
## 🔐 7. Authentication Management
### 7.1 User Management
```bash
# View authentication information
ceph auth list
ceph auth get client.admin
# Create user
ceph auth get-or-create client.<name> mon 'allow r' osd 'allow rw pool=<poolname>'
ceph auth get-or-create client.rbd mon 'profile rbd' osd 'profile rbd pool=rbd'
# Delete user
ceph auth del client.<name>
# Export/import keys
ceph auth export client.<name> > client.<name>.keyring
ceph auth import -i client.<name>.keyring
```
### 7.2 Permission Management
```bash
# Modify user permissions
ceph auth caps client.<name> mon 'allow r' osd 'allow rw pool=rbd'
# View permissions
ceph auth get client.<name>
# Generate client configuration
ceph config generate-minimal-conf
# Common permission templates
# Read-only user: mon 'allow r' osd 'allow r'
# RBD user: mon 'profile rbd' osd 'profile rbd pool=rbd'
# CephFS user: mon 'allow r' mds 'allow rw' osd 'allow rw pool=cephfs_data'
```
---
## 🏗️ 8. CRUSH Map Management
### 8.1 Basic CRUSH Operations
```bash
# View CRUSH map
ceph osd tree
ceph osd crush tree
# Export/import CRUSH map
ceph osd getcrushmap -o crushmap.bin
crushtool -d crushmap.bin -o crushmap.txt
crushtool -c crushmap.txt -o crushmap-new.bin
ceph osd setcrushmap -i crushmap-new.bin
# Create/delete bucket
ceph osd crush add-bucket <bucket-name> <bucket-type>
ceph osd crush remove <bucket-name>
# Move OSD to different location
ceph osd crush move osd.X host=<hostname>
ceph osd crush move osd.X rack=<rackname>
```
### 8.2 CRUSH Rule Management
```bash
# View CRUSH rules
ceph osd crush rule ls
ceph osd crush rule dump
# Create CRUSH rule
ceph osd crush rule create-simple <rule-name> <root> <type>
ceph osd crush rule create-replicated <rule-name> <root> <failure-domain> <device-class>
# Delete CRUSH rule
ceph osd crush rule rm <rule-name>
# Test CRUSH mapping
crushtool -i crushmap.bin --test --show-mappings --ruleset 0 --num-rep 3 --min-x 0 --max-x 100
```
---
## 🎯 9. RBD Management
### 9.1 Basic RBD Operations
```bash
# Create RBD image
rbd create --size 1024 <poolname>/<image-name>
rbd create --size 10G --image-feature layering,exclusive-lock mypool/myimage
# View RBD images
rbd ls <poolname>
rbd info <poolname>/<image-name>
rbd du <poolname>/<image-name>
# Delete RBD image
rbd rm <poolname>/<image-name>
rbd trash mv <poolname>/<image-name>    # Move to trash
rbd trash restore <poolname>/<image-id>  # Restore from trash
# Resize image
rbd resize --size 2048 <poolname>/<image-name>
rbd resize --size 2G <poolname>/<image-name>
# Create snapshot
rbd snap create <poolname>/<image-name>@<snap-name>
rbd snap ls <poolname>/<image-name>
rbd snap rm <poolname>/<image-name>@<snap-name>
```
### 9.2 Advanced RBD Operations
```bash
# Clone image
rbd snap protect <poolname>/<parent-image>@<snap-name>
rbd clone <poolname>/<parent-image>@<snap-name> <poolname>/<child-image>
# Flatten cloned image
rbd flatten <poolname>/<image-name>
# Export/import image
rbd export <poolname>/<image-name> image.raw
rbd export --export-format 2 <poolname>/<image-name> image.raw
rbd import image.raw <poolname>/<image-name>
# Image mapping
rbd map <poolname>/<image-name>
rbd unmap /dev/rbdX
rbd showmapped
# Image feature management
rbd feature enable <poolname>/<image-name> exclusive-lock
rbd feature disable <poolname>/<image-name> deep-flatten
```
---
## 🌐 10. CephFS Management
### 10.1 Basic CephFS Operations
```bash
# Create filesystem
ceph fs new <fs-name> <metadata-pool> <data-pool>
# View filesystem
ceph fs ls
ceph fs status <fs-name>
ceph fs get <fs-name>
# Delete filesystem
ceph fs fail <fs-name>
ceph fs rm <fs-name> --yes-i-really-mean-it
# MDS management
ceph mds stat
ceph mds dump
ceph mds fail <mds-id>
ceph mds repaired <mds-id>
```
### 10.2 CephFS Client
```bash
# Mount CephFS
mount -t ceph <mon-addr>:/ /mnt/cephfs -o name=admin,secret=<key>
mount -t ceph <mon-addr>:/ /mnt/cephfs -o name=admin,secretfile=/etc/ceph/admin.secret
# Use ceph-fuse
ceph-fuse /mnt/cephfs
ceph-fuse /mnt/cephfs -o allow_other
# View client connections
ceph daemon mds.<id> client ls
ceph daemon mds.<id> session ls
# Evict client
ceph daemon mds.<id> client evict id=<client-id>
```
### 10.3 Advanced CephFS Management
```bash
# Subdirectory mounting
mount -t ceph <mon-addr>:/subdir /mnt/subdir -o name=admin
# Configure multiple filesystems
ceph fs flag set enable_multiple true
# Directory quotas
setfattr -n ceph.quota.max_files -v 10000 /mnt/cephfs/dir
setfattr -n ceph.quota.max_bytes -v 1000000000 /mnt/cephfs/dir
getfattr -n ceph.quota.max_files /mnt/cephfs/dir
```
---
## ☁️ 11. RGW Management
### 11.1 Basic RGW Operations
```bash
# Create user
radosgw-admin user create --uid=<user-id> --display-name="<display-name>"
radosgw-admin user create --uid=testuser --display-name="Test User" --email=test@example.com
# View users
radosgw-admin user list
radosgw-admin user info --uid=<user-id>
# Delete user
radosgw-admin user rm --uid=<user-id>
# Modify user
radosgw-admin user modify --uid=<user-id> --display-name="New Name"
# Create subuser
radosgw-admin subuser create --uid=<user-id> --subuser=<subuser-id> --access=full
```
### 11.2 RGW Key Management
```bash
# Create access key
radosgw-admin key create --uid=<user-id> --key-type=s3
# Delete access key
radosgw-admin key rm --uid=<user-id> --key-type=s3 --access-key=<access-key>
# Generate Swift key
radosgw-admin key create --uid=<user-id> --key-type=swift --gen-secret
```
### 11.3 RGW Maintenance
```bash
# Check bucket index
radosgw-admin bucket check --bucket=<bucket-name>
# Rebuild bucket index
radosgw-admin bucket reshard --bucket=<bucket-name> --num-shards=<num>
# Garbage collection
radosgw-admin gc list
radosgw-admin gc process
# View usage statistics
radosgw-admin usage show --uid=<user-id>
radosgw-admin usage show --show-log-entries=false
# Bucket management
radosgw-admin bucket list
radosgw-admin bucket stats --bucket=<bucket-name>
```
---
## 🔧 12. Configuration Management
### 12.1 Runtime Configuration
```bash
# View configuration
ceph config dump
ceph config get <daemon-type>.<daemon-id> <option>
ceph config show <daemon-type>.<daemon-id>
# Set configuration
ceph config set <daemon-type> <option> <value>
ceph config set global osd_pool_default_size 3
ceph config set osd osd_max_backfills 1
# Reset configuration
ceph config rm <daemon-type> <option>
# View configuration history
ceph config log
```
### 12.2 Temporary Configuration Adjustment
```bash
# Temporary configuration (lost after restart)
ceph tell <daemon-type>.<id> config set <option> <value>
ceph tell osd.* config set debug_osd 10
# View configuration
ceph tell <daemon-type>.<id> config show
ceph tell <daemon-type>.<id> config get <option>
# Batch configuration
ceph tell osd.* config set osd_max_backfills 1
ceph tell mon.* config set debug_mon 5
```
---
## 📈 13. Performance Analysis
### 13.1 RBD Performance Analysis
#### RBD Performance Testing Tools
```bash
# Use rbd bench for performance testing
rbd bench --io-type write --io-size 4K --io-threads 16 --io-total 1G <pool>/<image>
rbd bench --io-type read --io-size 4K --io-threads 16 --io-total 1G <pool>/<image>
rbd bench --io-type readwrite --io-size 4K --io-threads 16 --io-total 1G <pool>/<image>
# Use fio for detailed performance testing
fio --name=rbd-test --ioengine=rbd --pool=<pool> --rbdname=<image> \
    --rw=randwrite --bs=4k --iodepth=32 --numjobs=4 --runtime=60 --group_reporting
# Test different block sizes
for bs in 4k 8k 16k 32k 64k 128k; do
    echo "Testing block size: $bs"
    rbd bench --io-type write --io-size $bs --io-threads 16 --io-total 512M <pool>/<image>
done
# Sequential read/write test
fio --name=seq-write --ioengine=rbd --pool=test --rbdname=testimg \
    --rw=write --bs=1M --iodepth=1 --numjobs=1 --runtime=60
```
#### RBD Performance Monitoring
```bash
# Real-time RBD I/O monitoring
rbd perf image iostat
# View RBD image statistics
rbd du <pool>/<image>
rbd info <pool>/<image>
# Monitor client I/O patterns
ceph daemon client.<id> perf dump
ceph daemon client.<id> perf histogram dump
# View RBD cache statistics
ceph daemon client.<id> config show | grep rbd_cache
ceph daemon client.<id> perf dump | grep rbd_cache
```
#### RBD Performance Tuning Parameters
```bash
# Client cache optimization
ceph config set client rbd_cache true
ceph config set client rbd_cache_size 134217728           # 128MB
ceph config set client rbd_cache_max_dirty 100663296      # 96MB
ceph config set client rbd_cache_target_dirty 67108864    # 64MB
ceph config set client rbd_cache_max_dirty_age 10.0
# Readahead optimization
ceph config set client rbd_readahead_trigger_requests 10
ceph config set client rbd_readahead_max_bytes 524288     # 512KB
ceph config set client rbd_readahead_disable_after_bytes 52428800  # 50MB
# Concurrency optimization
ceph config set client rbd_concurrent_management_ops 20
ceph config set client rbd_op_threads 1
```
### 13.2 CephFS Performance Analysis
#### CephFS Performance Testing
```bash
# Basic performance testing with dd
dd if=/dev/zero of=/mnt/cephfs/testfile bs=1M count=1000 oflag=direct
dd if=/mnt/cephfs/testfile of=/dev/null bs=1M iflag=direct
# Comprehensive performance testing with iozone
iozone -a -s 1G -r 4k -r 64k -r 1M -i 0 -i 1 -i 2 -f /mnt/cephfs/testfile
# Small file performance test
mkdir /mnt/cephfs/small_files_test
time for i in {1..10000}; do touch /mnt/cephfs/small_files_test/file_$i; done
# Metadata performance test
time find /mnt/cephfs -name "*.txt" | wc -l
time ls -la /mnt/cephfs/large_directory/ | wc -l
# Concurrent file operations test
for i in {1..10}; do
    (for j in {1..1000}; do
        touch /mnt/cephfs/test_${i}_${j}
    done) &
done
wait
```
#### CephFS Performance Monitoring
```bash
# MDS performance statistics
ceph daemon mds.<id> perf dump
ceph daemon mds.<id> session ls
ceph daemon mds.<id> status
ceph daemon mds.<id> ops
# Client performance statistics
ceph daemon client.<id> perf dump
ceph daemon client.<id> client_metadata
# View MDS load
ceph fs status <fsname>
ceph mds metadata <mds-id>
# Monitor directory fragmentation
ceph daemon mds.<id> dirfrag ls <path>
ceph daemon mds.<id> dirfrag split_info <path>
```
### 13.3 OSD Performance Analysis
#### OSD Performance Benchmarking
```bash
# OSD benchmark testing
ceph tell osd.<id> bench 1073741824 4194304              # 1GB test, 4MB block size
# Direct ceph-osd testing
ceph-osd -i <id> --test-objectstore --test-objectstore-workload uniform_random
# Disk performance testing
dd if=/dev/zero of=/var/lib/ceph/osd/ceph-<id>/test bs=1M count=1000 oflag=direct
dd if=/var/lib/ceph/osd/ceph-<id>/test of=/dev/null bs=1M iflag=direct
# Network performance testing
iperf3 -s    # Run on target OSD node
iperf3 -c <target-osd-ip> -t 30 -P 4    # Run on source node
# Use rados bench for testing
rados bench -p <pool> 60 write --no-cleanup
rados bench -p <pool> 60 seq
rados bench -p <pool> 60 rand
rados -p <pool> cleanup
```
#### Detailed OSD Performance Monitoring
```bash
# OSD performance counters
ceph daemon osd.<id> perf dump
ceph daemon osd.<id> perf histogram dump
# OSD operation statistics
ceph daemon osd.<id> dump_ops_in_flight
ceph daemon osd.<id> dump_blocked_ops
ceph daemon osd.<id> dump_op_pq_state
# OSD storage statistics
ceph daemon osd.<id> status
ceph daemon osd.<id> df
# Slow operation analysis
ceph daemon osd.<id> dump_slow_requests
ceph daemon osd.<id> dump_historic_slow_ops
# OSD recovery statistics
ceph daemon osd.<id> dump_recovery_reservations
ceph daemon osd.<id> dump_backfill_reservations
```
### 13.4 Overall Cluster Performance Analysis
#### Cluster-level Performance Monitoring
```bash
# Cluster I/O statistics
ceph iostat
watch "ceph iostat"
# PG distribution analysis
ceph pg dump | grep ^pg | awk '{print $1,$15}' | sort
# Real-time performance monitoring script
#!/bin/bash
while true; do
    echo "$(date) - IOPS: $(ceph iostat | grep -E 'read|write')"
    echo "$(date) - Health: $(ceph health)"
    sleep 5
done
# Cluster latency analysis
ceph osd perf | grep -E "apply_latency|commit_latency"
```
---
## 🛠️ 14. Ceph Specialized Tools
### 14.1 Data Recovery and Diagnostic Tools
#### ceph-objectstore-tool (Object Store Tool)
```bash
# List object store contents
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id> --op list
# Export PG data
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id> \
    --op export --pgid <pg-id> --file /tmp/pg_export.dat
# Import PG data
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id> \
    --op import --file /tmp/pg_export.dat
# Remove corrupted object
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id> \
    --op remove --pgid <pg-id> --oid <object-id>
# Repair object store
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id> \
    --op fsck --debug
# View object information
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-<id> \
    --op info --pgid <pg-id> --oid <object-id>
```
#### ceph-kvstore-tool (Key-Value Store Tool)
```bash
# List key-value pairs
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-<id> list
# Get specific key-value
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-<id> get <prefix> <key>
# Delete key-value pair
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-<id> rm <prefix> <key>
# Compact database
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-<id> compact
# Backup/restore key-value store
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-<id> backup /tmp/kv_backup
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-<id> restore /tmp/kv_backup
# Statistics
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-<id> stats
```
#### ceph-bluestore-tool (BlueStore Specific Tool)
```bash
# View BlueStore information
ceph-bluestore-tool show-label --dev /dev/sdX
ceph-bluestore-tool prime-osd-dir --dev /dev/sdX --path /var/lib/ceph/osd/ceph-X
# Repair BlueStore
ceph-bluestore-tool fsck --path /var/lib/ceph/osd/ceph-<id>
ceph-bluestore-tool repair --path /var/lib/ceph/osd/ceph-<id>
# Show BlueStore metadata
ceph-bluestore-tool show-label --dev /dev/sdX
ceph-bluestore-tool show-label --path /var/lib/ceph/osd/ceph-<id>
# BlueStore space analysis
ceph-bluestore-tool bluefs-stats --path /var/lib/ceph/osd/ceph-<id>
ceph-bluestore-tool bluefs-bdev-sizes --path /var/lib/ceph/osd/ceph-<id>
# Allocator tools
ceph-bluestore-tool free-dump --path /var/lib/ceph/osd/ceph-<id>
ceph-bluestore-tool free-score --path /var/lib/ceph/osd/ceph-<id>
```
### 14.2 Monitoring and Management Tools
#### crushtool (CRUSH Mapping Tool)
```bash
# Compile/decompile CRUSH map
ceph osd getcrushmap -o crushmap.bin
crushtool -d crushmap.bin -o crushmap.txt
crushtool -c crushmap.txt -o crushmap_new.bin
ceph osd setcrushmap -i crushmap_new.bin
# Test CRUSH rules
crushtool -i crushmap.bin --test --show-mappings --ruleset 0 --num-rep 3 --min-x 0 --max-x 100
# Analyze CRUSH weights
crushtool -i crushmap.bin --show-choose-tries
# Verify CRUSH map
crushtool -i crushmap.bin --check
# Show CRUSH tree structure
crushtool -i crushmap.bin --tree
```
#### monmaptool (Monitor Mapping Tool)
```bash
# Create Monitor map
monmaptool --create --add <mon-id> <mon-addr> --fsid <uuid> monmap.bin
# View Monitor map
monmaptool --print monmap.bin
# Add/remove Monitor
monmaptool --add <mon-id> <mon-addr> monmap.bin
monmaptool --rm <mon-id> monmap.bin
# Generate new fsid
uuidgen
monmaptool --create --fsid $(uuidgen) monmap.bin
# Extract from existing cluster
ceph mon getmap -o monmap.bin
monmaptool --print monmap.bin
```
#### osdmaptool (OSD Mapping Tool)
```bash
# Export OSD map
ceph osd getmap -o osdmap.bin
osdmaptool osdmap.bin --print
# Test PG mapping
osdmaptool osdmap.bin --test-map-pgs --pool <pool-id>
# Create incremental map
osdmaptool --createsimple 3 osdmap.bin --with-default-pool
# Export specific epoch map
ceph osd getmap <epoch> -o osdmap_<epoch>.bin
# Show OSD tree
osdmaptool osdmap.bin --tree
```
### 14.3 Data Migration and Synchronization Tools
#### rados (Object Storage Client)
```bash
# Basic object operations
rados -p <pool> put <object> <file>
rados -p <pool> get <object> <file>
rados -p <pool> ls
rados -p <pool> rm <object>
# Batch operations
rados -p <pool> ls | xargs -I {} rados -p <pool> rm {}
# Benchmark testing
rados bench -p <pool> 60 write --no-cleanup
rados bench -p <pool> 60 seq
rados bench -p <pool> 60 rand
# Cleanup benchmark data
rados -p <pool> cleanup
# Object attribute operations
rados -p <pool> setxattr <object> <attr> <value>
rados -p <pool> getxattr <object> <attr>
rados -p <pool> listxattr <object>
# Watch object changes
rados -p <pool> watch <object>
```
#### rbd-mirror (RBD Mirror Tool)
```bash
# Enable mirroring
rbd mirror pool enable <pool> pool
rbd mirror pool enable <pool> image
# Configure mirroring
rbd mirror image enable <pool>/<image> snapshot
rbd mirror image enable <pool>/<image> journal
# View mirror status
rbd mirror pool status <pool>
rbd mirror image status <pool>/<image>
# Force resync
rbd mirror image resync <pool>/<image>
# Failover
rbd mirror image promote <pool>/<image>
rbd mirror image demote <pool>/<image>
```
#### ceph-crash (Crash Reporting Tool)
```bash
# View crash reports
ceph crash ls
ceph crash info <crash-id>
# Archive crash reports
ceph crash archive <crash-id>
ceph crash archive-all
# Generate crash report
ceph crash post -i <crash-dump-file>
# Crash statistics
ceph crash stat
```
#### ceph-volume (Volume Management Tool)
```bash
# Create OSD with LVM
ceph-volume lvm create --data /dev/sdb
ceph-volume lvm create --data /dev/sdb --block.wal /dev/nvme0n1p1
# Separate prepare and activate
ceph-volume lvm prepare --data /dev/sdb --osd-id 0
ceph-volume lvm activate 0 <osd-uuid>
# View LVM information
ceph-volume lvm list
ceph-volume lvm list /dev/sdb
# Simple method to create OSD
ceph-volume simple scan /dev/sdb
ceph-volume simple activate <osd-id> <osd-uuid>
```
---
## 🚨 15. Troubleshooting
### 15.1 Log Viewing
```bash
# View service logs
journalctl -u ceph-mon@<hostname> -f
journalctl -u ceph-osd@<id> -f
journalctl -u ceph-mds@<hostname> -f
journalctl -u ceph-mgr@<hostname> -f
# View logs by time range
journalctl -u ceph-osd@<id> --since "2023-01-01 00:00:00" --until "2023-01-01 23:59:59"
# Adjust log levels
ceph tell osd.* config set debug_osd 10
ceph tell mon.* config set debug_mon 10
ceph tell mds.* config set debug_mds 10
# View real-time errors
tail -f /var/log/ceph/ceph.log | grep -i error
tail -f /var/log/ceph/ceph-osd.*.log | grep -i slow
```
### 15.2 Common Issue Troubleshooting
```bash
# PG inconsistency issues
ceph health detail | grep inconsistent
ceph pg <pgid> query
ceph pg repair <pgid>
# OSD full issues
ceph osd df | grep -E "(95|96|97|98|99)%"
ceph osd reweight-by-utilization
# Slow request issues
ceph daemon osd.<id> dump_slow_requests
ceph daemon osd.<id> dump_historic_slow_ops
ceph config set osd debug_osd 10
# Clock synchronization issues
ceph time-sync-status
ntpq -p
# Disk space issues
ceph df
ceph osd df
df -h /var/lib/ceph/osd/
```
---
## 🔄 16. Backup and Recovery
### 16.1 Data Backup
```bash
# Export Monitor map
ceph mon getmap -o monmap.bin
# Export OSD map
ceph osd getmap -o osdmap.bin
# Export CRUSH map
ceph osd getcrushmap -o crushmap.bin
# Backup ceph.conf and keys
cp /etc/ceph/ceph.conf /backup/
cp /etc/ceph/ceph.client.admin.keyring /backup/
# Backup authentication information
ceph auth export > /backup/ceph_auth.txt
# Backup pool information
ceph osd pool ls detail > /backup/pools.txt
ceph pg dump > /backup/pg_dump.txt
```
### 16.2 Disaster Recovery
```bash
# Recover data from single OSD
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-X \
    --export-remove --pgid <pgid> --file /tmp/pg_export
# Rebuild Monitor
ceph-mon --mkfs -i <mon-id> --monmap monmap.bin --keyring mon.keyring
# Restore authentication information
ceph auth import -i /backup/ceph_auth.txt
# Restore CRUSH map
ceph osd setcrushmap -i /backup/crushmap.bin
# OSD data recovery
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-X \
    --op import --file /tmp/pg_export
```
### 16.3 RBD Backup and Recovery
```bash
# RBD snapshot backup
rbd snap create <pool>/<image>@backup-$(date +%Y%m%d)
rbd export <pool>/<image>@<snapshot> /backup/image_backup.raw
# RBD incremental backup
rbd export-diff <pool>/<image>@<snap1> <pool>/<image>@<snap2> /backup/diff.raw
# RBD recovery
rbd import /backup/image_backup.raw <pool>/<new-image>
rbd import-diff /backup/diff.raw <pool>/<image>
```
---
**This document is continuously updated. Please check for the latest version regularly. Any questions or suggestions are welcome.**
*Last updated: June 2025*
