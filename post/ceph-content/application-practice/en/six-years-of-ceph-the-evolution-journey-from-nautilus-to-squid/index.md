

This document provides a detailed analysis of the major feature evolution in Ceph from Nautilus (v14) to the latest Squid (v19) versions, offering guidance for selecting appropriate versions and developing upgrade strategies. (Organized with LLM assistance)

## Version Overview

| Version | Codename | Release Date | Lifecycle Status |
|---------|----------|--------------|------------------|
| v14.2.x | Nautilus | 2019 | EOL |
| v15.2.x | Octopus | 2020 | EOL |
| v16.2.x | Pacific | 2021 | EOL |
| v17.2.x | Quincy | 2022 | EOL |
| v18.2.x | Reef | 2023 | Stable Maintenance |
| v19.2.x | Squid | 2024 | Current Stable |

---

## Version Feature Comparison Summary

### Maturity-Driven Version Selection Guide

| Feature | Nautilus | Octopus | Pacific | Quincy | Reef | Squid |
|---------|----------|---------|---------|---------|------|-------|
| **Deployment Method** | ceph-deploy | **cephadm introduced** | cephadm | cephadm mature | cephadm | cephadm |
| **Storage Engine** | BlueStore | BlueStore | BlueStore | BlueStore | **FileStore removed** | BlueStore optimized |
| **Configuration Management** | **Centralized introduced** | Centralized | Centralized | Centralized | Centralized | Centralized |
| **Network Protocol** | **msgr2 introduced** | msgr2 stable | msgr2 | msgr2 | msgr2 | msgr2 |
| **PG Management** | **autoscale introduced** | autoscale | autoscale | autoscale | autoscale | autoscale |
| **Scheduler** | Traditional | Improved | **mclock introduced** | **mclock default** | mclock | mclock optimized |
| **CephFS Multi-FS** | **First support** | Feature enhanced | **Mirroring perfected** | Management optimized | Management optimized | Dashboard integrated |
| **Multi-site** | Basic | **RBD mirroring** | **CephFS mirroring** | Perfected | Enhanced | Enhanced |
| **Dashboard** | Basic | Improved | Improved | Improved | **Refactored** | **Refactored** |
| **Containerization** | None | **cephadm preview** | **cephadm mature** | cephadm complete | cephadm | cephadm |

#### Feature Maturity Marking Legend
- **Bold text**: Important milestones for features in this version (first introduction/stability achieved/major improvements)
- Normal text: Features remain stable or have minor improvements in this version
- Italic text: Features are deprecated or being prepared for removal in this version

### Feature-Driven Version Selection Guide

| Required Feature | Minimum Version | Stable Recommended Version | Notes |
|------------------|-----------------|----------------------------|-------|
| Centralized Configuration Management | Nautilus | Octopus+ | Basic functionality available, upgrade recommended for stability |
| PG autoscaling | Nautilus | Pacific+ | Production environments recommend manual control |
| msgr2 Security Protocol | Nautilus | Octopus+ | Recommended for new deployments |
| cephadm Container Management | Octopus | Pacific+ | Tech preview → production ready |
| CephFS Multi-filesystem | Nautilus | Pacific+ | Basic support → production ready |
| CephFS Mirroring/DR | Octopus | Pacific+ | Feature introduction → production stable |
| mclock QoS Scheduling | Pacific | Quincy+ | Introduction → default enabled |
| Full Containerized Deployment | Pacific | Quincy+ | Basic support → complete ecosystem |
| Advanced Dashboard | Reef | Squid+ | Refactored → complete |
| FileStore Replacement | Any version | Reef+ | FileStore support removed after Reef |

## Feature First Introduction and Maturity Analysis

### Important Feature Lifecycle Timeline

This section details the first introduction version, stable version, and recommended production environment adoption timing for key features, helping users make informed version choices.

#### Configuration Management Evolution
- **Centralized Configuration Management**
  - **First introduced**: Nautilus (v14.x) - Basic functionality available
  - **Stable recommended**: Octopus (v15.x) - Fixed critical bugs, production ready
  - **Technical value**: Unified configuration management, supports online modification, provides audit functionality

#### Storage Pool Management Revolution
- **PG autoscaling/merging**
  - **First introduced**: Nautilus (v14.x) - Historical breakthrough, supports PG count reduction
  - **Stable recommended**: Pacific (v16.x) - Algorithm optimization, reduced misoperation risks
  - **⚠️ Production advice**: Manual control recommended to avoid I/O impact from automatic tuning

#### Network Protocol Upgrade
- **msgr2 protocol**
  - **First introduced**: Nautilus (v14.x) - Significant security and performance improvements
  - **Stable recommended**: Octopus (v15.x) - Fixed compatibility issues, suitable for large-scale deployment
  - **Technical value**: End-to-end encryption, stronger authentication, lower latency

#### Orchestration Management Revolution
- **cephadm containerized orchestration**
  - **First introduced**: Octopus (v15.x) - Technical preview version
  - **Stable recommended**: Pacific (v16.x) - Production ready, feature complete
  - **Full maturity**: Quincy (v17.x) - Replaces ceph-deploy, becomes mainstream solution
  - **Revolutionary significance**: Transition from package management to containerization, greatly simplifying operations

#### Distributed File System Evolution
- **CephFS Multi-filesystem**
  - **First introduced**: Nautilus (v14.x) - Basic multi-FS support
  - **Feature enhancement**: Octopus (v15.x) - Snapshot and mirroring features
  - **Production recommended**: Pacific (v16.x) - Mirroring feature stable, suitable for production environments
  - **Management optimization**: Squid (v19.x) - Dashboard integration, subvolume management improvements

#### Storage Engine Development
- **BlueStore maturation**
  - **Default engine**: Mimic (v13.x) - BlueStore replaces FileStore as default storage engine
  - **Performance optimization**: Nautilus (v14.x) - bitmap allocator, significant performance improvements
  - **Full maturity**: Pacific (v16.x) - Optimal performance and stability
  - **FileStore retirement**: Reef (v18.x) - FileStore support completely removed

- **Crimson/Seastore (Next Generation)**
  - **Technical preview**: Quincy (v17.x) - Experimental features
  - **Feature enhancement**: Squid (v19.x) - Supports RBD workloads
  - **Expected maturity**: Future versions - Production environment ready

## Nautilus (N Version)

### Release Status
- **Current version**: v14.2.22
- **Version series**: v14.2.0 to v14.2.22
- **Lifecycle**: v14.2.22 is the last version of the Nautilus series, EOL

### Main Highlights

#### Centralized Configuration Management
- **Feature**: Compared to the previous approach where all persistent configurations needed to be written in ceph.conf, the current approach is more convenient for management, including viewing, modification, and auditing
- **Core value**: Unified management of all cluster configurations, supports online modification, provides configuration history and audit functionality, greatly simplifying configuration management for large-scale clusters

##### Configuration Management Commands
```bash
# View existing config settings
ceph config dump

# get/set configurations for a specific level or instance, supports multi-level override with finest granularity
ceph config get $level $entityname
ceph config set $level $entityname

# Examples
ceph config set global osd_recovery_sleep_hdd 0.1
ceph config set osd osd_recovery_sleep_hdd 0.1
ceph config set 'osd.*' osd_recovery_sleep_hdd  # Service level requires *
ceph config set osd.1 osd_recovery_sleep_hdd 0.2

# View help information for an entity
sudo ceph config help osd_recovery_sleep_hdd

# Modification history log for auditing
sudo ceph config log
```

##### Configuration Management Key Points
- Some parameters that previously required restart can now be adjusted online, can be confirmed via help, such as osd_target_memory
- osd_class_update_on_start needs to be set to true when creating OSD, will automatically add to tree, then set to false to prevent crush tree changes on OSD restart

#### PG Merging and Autotuning
- **⭐ Major breakthrough**: This is a major breakthrough in Ceph history, Nautilus first supports PG count reduction (merging) and auto-tuning functionality
- **PG count reduction**: PG count can be reduced, meaning merging is now possible, solving the previous limitation where PG count could only increase
- **PG default autoscale**: PG autoscaling is enabled by default, automatically adjusting PG count based on data volume
- **Technical significance**: Makes storage pool management more intelligent and flexible, reducing manual tuning workload
- **⚠️ Production environment note**: Adjustments cause peering process, resulting in I/O blocking and large data migration. Production environments recommend setting `osd_pool_default_pg_autoscale_mode=off`, manually controlling PG adjustment timing

#### Device Management and Failure Prediction
- **Device management**: New device management functionality
- **Failure prediction**: Failure prediction model based on device health metrics

#### Crash Information Collection
- **Service**: A ceph-crash service runs on each node
- **Functionality**: If a process crashes (e.g., assert), collects environment information and backtrace, and reports it

#### Messaging (msgr2) and Security Configuration
- **⭐ Protocol revolution**: Nautilus first introduces msgr2 protocol, a major upgrade to Ceph network communication
- **msgr2 protocol**: New messaging protocol v2, providing better security and performance
  - **Encrypted transmission**: Supports end-to-end encryption, protecting internal cluster communication security
  - **Authentication**: Provides stronger authentication mechanisms, preventing man-in-the-middle attacks
  - **Performance improvement**: Optimized network protocol stack, reducing latency and CPU overhead
- **Backward compatibility**: To ensure compatibility with old clients, needs configuration to support both msgr1 and msgr2
- **Main configuration**: global_id and not enabling msgr2 (if compatibility with old environments needed)
- **Migration advice**: New deployments recommend direct use of msgr2, old clusters can migrate gradually

#### RADOS Related Improvements
- **bluestore**: New 'bitmap' allocator (better performance than avl allocator, reduces fragmentation)
- **Asynchronous recovery**: Improved async recovery (reduces recovery impact on client I/O)
- **scrub**: Scrub supports autorepair (automatically fixes detected data inconsistency issues)
- **Erasure coding**: Added new erasure coding algorithms (provides better storage efficiency and fault tolerance)
- **Pool quotas**: Pools support setting quotas (limits storage pool space and object count usage)
- **Load balancing**: ceph balancer (automatically optimizes data distribution, improves cluster performance balance)

#### RBD Related Improvements
- **Namespaces**: Supports namespaces for multi-tenant resource isolation, different tenants can use same image names without conflicts
- **Timestamps**: Volumes support creation, access, modification timestamps for auditing and management
- **Monitoring enhancement**: mgr's Prometheus plugin can now monitor detailed I/O statistics for individual volumes, providing more granular performance analysis capabilities

#### CephFS Related Improvements
- **Multi-filesystem**: Supports creating and managing multiple independent filesystems in a single cluster, each with independent metadata pools
  - **⭐ First introduction**: Nautilus version first introduces multi-filesystem support, laying foundation for future version improvements
  - **Stability advice**: Although functionality is available, recommend thorough testing in production environments first
- **Management interface**: Introduces volume management concept, abstracting complex filesystem operations into simple volume management, simulating block storage management approach
  - **Concept mapping**: fs(filesystem) → volume, directory → subvolume
- **Stability improvements**: 
  - **Client caps limitation**: Limits client caps acquisition speed, preventing MDS memory overflow (OOM) due to excessive caps requests
  - **Caps**: CephFS capability authorization mechanism, controlling client access permissions to files/directories
- **Startup optimization**: 
  - **OpenTable files**: Records information about opened files, if MDS restart is too slow, can delete this file to accelerate startup
  - **Applicable scenarios**: Particularly useful in emergency recovery or environments with many open files
- **Quota functionality**: Kernel clients now support quota limits, can set space and file count limits for directories or subvolumes

##### Subvolume
- **Subvolume management**: Supports creating isolated subvolumes within filesystems, each with independent quotas and access control
- **CSI integration**: If using Kubernetes CSI driver, need to create dedicated CSI subvolume groups to manage dynamic volume provisioning

##### LAZYIO
- **Lazy IO mode**: New IO mode allowing applications to delay write operations, improving small file write performance
- **Caching mechanism**: Clients can cache write data, reducing network round trips
- **Applicable scenarios**: Particularly suitable for frequent small file write workloads

#### ceph-mgr
- **Prometheus plugin**: 
  - **Standard monitoring**: mgr's prometheus plugin provides standard monitoring metrics interface, requires manual enabling
  - **Integration advantages**: Seamless integration with monitoring systems like Grafana, providing rich performance dashboards
- **Stability issues**: 
  - **CephFS scenarios**: In CephFS environments, mgr may experience deadlock issues, requiring periodic mgr service restarts
  - **Monitoring advice**: Recommend monitoring mgr service status, restart promptly when abnormalities occur
- **Module management**: 
  - **Restful module bug**: Restful module has memory leak issues, recommend disabling this module to avoid mgr OOM
  - **Disable command**: `ceph mgr module disable restful`

### Important Updates and Fixes

#### Performance Optimization
- **BlueStore cache**: `bluefs_buffered_io` default set to true, improving metadata-intensive workload performance
- **Memory management**: `bluestore_cache_trim_max_skip_pinned` default value increased to 1000, controlling memory growth caused by onodes
- **Monitor trimming**: Introduces dynamic trimming logic, controlled by `paxos_service_trim_max_multiplier` parameter

#### Stability Improvements
- **BlueStore fixes**: Fixed unexpected ENOSPC bug in Avl/Hybrid allocators
- **Client interoperability**: Fixed long-standing bug preventing 32-bit and 64-bit clients from interoperating under msgr v2
- **OSD downtime**: Added `--max <n>` option for `osd ok-to-stop` command, providing number of OSDs that can be stopped simultaneously

#### New Features
- **⭐ OSD shutdown notification**: First introduces `osd_fast_shutdown_notify_mon` option, allowing OSDs to notify monitors during fast shutdown
  - **Technical value**: Improves cluster management, reduces unnecessary OSD failure alerts
  - **Future improvements**: This feature is further optimized in Pacific version
- **Multi-architecture support**: Fixed issues with armv7l (armhf) mixed deployment with x86_64 or aarch64 servers

#### Deployment Notes
- **Tools**: Currently still uses ceph-deploy for deployment, this is the last version supported by ceph-deploy
- **Requirements**: ceph-deploy requires python2.7 and needs code modifications to support debian11

---

## Octopus

### Release Status
- **Current version**: v15.2.17
- **Version series**: v15.2.0 to v15.2.17
- **Lifecycle**: v15.2.17 is the last version of the Octopus series, archived

### Core Themes
Ceph Octopus version focuses on five different themes: multi-site usage, quality, performance, usability, and ecosystem.

### Main Highlights

#### Multi-site Functionality
- **CephFS snapshots**: 
  - **Snapshot scheduling**: Automatic timed snapshot creation, supports different cycles like hourly, daily, weekly, monthly
  - **Snapshot pruning**: Automatic cleanup of expired snapshots, avoiding unlimited storage space growth
  - **Regular snapshot automation**: Flexible snapshot policy configuration through cron expressions
- **Remote synchronization**: 
  - **Cross-cluster sync**: Supports syncing data to remote Ceph clusters for offsite backup
  - **Incremental sync**: Only syncs changed data, greatly reducing network bandwidth usage
- **RBD mirroring**: 
  - **Snapshot mirroring mode**: New snapshot-based mirroring mode, providing more reliable data consistency
  - **Asynchronous replication**: Supports asynchronous replication, reducing impact on primary cluster performance
- **Application value**: 
  - **Automatic backup**: Automated backup strategies without manual intervention
  - **Storage optimization**: Incremental backup significantly saves storage space
  - **Data protection**: Provides multi-layered data protection and disaster recovery capabilities

#### Quality Improvements
- **Crash alerts**: Ceph daemon crashes trigger simple health alerts and can trigger email sending
- **Device health**: New "device" telemetry channel improves device failure prediction models
- **Telemetry**: Users can opt into expanded telemetry content

#### Performance Optimization
- **Recovery optimization**: Recovery tail latency has been improved, can synchronize objects during recovery by copying only object deltas
- **BlueStore improvements**: 
  - Improved per-pool calculation of "omap" (key/value) object data
  - Improved cache memory management
  - Reduced allocation unit size for SSD devices

#### Usability Enhancement
- **⭐ cephadm orchestrator**: 
  - **🎯 First introduction**: Octopus version first introduces cephadm orchestrator, marking a revolutionary change in Ceph operations management
  - **Management revolution**: Transition from traditional package-based management to modern containerized orchestration management, greatly reducing deployment and operations complexity
  - **Containerized deployment**: All Ceph components run in containers, providing better isolation and maintainability
  - **Automated management**: Automatically handles service discovery, health checks, failure recovery, and other operational tasks
  - **Maturity**: Technical preview in Octopus, reaches production-ready maturity after Pacific version
- **Orchestrator API**: 
  - **Unified interface**: Provides unified API interface for interacting with cephadm orchestrator
  - **Declarative management**: Supports declarative service configuration, simplifying cluster management
- **SSH management**: 
  - **Agentless architecture**: Manages each node through SSH connections without installing additional agents on each node
  - **Secure connections**: Uses SSH key authentication to ensure management connection security
  - **Command execution**: Supports executing management commands on remote nodes for centralized management
- **Dashboard integration**: 
  - **Visual management**: Ceph Dashboard deeply integrated with orchestrator, providing graphical cluster management interface
  - **Service management**: Can directly manage service start/stop, upgrades through web interface
- **Health alerts**: 
  - **Alert management**: Supports temporarily or permanently silencing specific health alerts
  - **Noise reduction**: Avoids non-critical alert interference, focuses on truly important issues
- **Historical significance**: cephadm marks Ceph's major transition from traditional package-based deployment to modern containerized orchestration management, greatly reducing Ceph cluster deployment, upgrade, and daily operations complexity

#### Ecosystem Enhancement
- **ceph-csi**: Now supports RWO and RWX through RBD and CephFS
- **Rook integration**: Integration with Rook provides turnkey ceph-csi by default
- **Feature support**: Includes RBD mirroring, RGW multi-site, and will eventually include CephFS mirroring

### Important Fixes and Security Updates

#### Security Vulnerability Fixes
- **CVE-2022-0670**: Fixed path restriction bypass vulnerability in OpenStack Manila Native-CephFS
- **Impact scope**: Only affects clusters using OpenStack Manila to provide native CephFS access
- **Fix content**: Fixed bug in Ceph Manager "volumes" plugin responsible for managing CephFS subvolumes

#### Critical Bug Fixes
- **SnapMapper key format**: Fixed SnapMapper key format conversion bug introduced in v15.2.x
- **RBD performance**: Fixed functionality of `rbd perf image iostat` and `rbd perf image iotop` commands
- **Stability**: Multiple stability and performance improvements

---

## Pacific

### Release Status
- **Current version**: v16.2.15
- **Version series**: v16.2.0 to v16.2.15
- **Lifecycle**: v16.2.15 is the last version of the Pacific series, archived

### Main Highlights

#### Management Simplification
- **Configuration anomaly detection**: New capability to detect configuration anomalies, helping identify and resolve configuration issues
- **Cluster upgrade monitoring**: New health check `DAEMON_OLD_VERSION` warns when different versions of Ceph are running on daemons
- **Progress module**: MGR progress module can now be toggled with `ceph progress on` and `ceph progress off` commands

#### Storage Backend Optimization
- **ceph-volume rewrite**: `lvm batch` subcommand underwent major rewrite, fixing many bugs and improving usability
- **OSD improvements**: 
  - New `osd_compact_on_start` configuration option triggers OSD compaction on startup (compressing storage engine database to reduce space usage)
  - Improved OSD fast shutdown mechanism, optimized based on Nautilus introduced `osd_fast_shutdown_notify_mon` feature, improving monitor communication efficiency
- **BlueStore enhancements**: Multiple BlueStore improvements and performance optimizations

#### Scheduler Improvements
- **mclock scheduler**: 
  - **QoS scheduling**: Refined mclock scheduler based on mClock algorithm for precise service quality control
  - **Working principle**: Precisely controls resource allocation for different operation types through reservation, limitation, and weight parameters
  - **Built-in profiles**: Provides three preset configurations, users can choose based on business needs:
    - **high_client_ops** (default): Allocates more OSD bandwidth to external client operations, prioritizing business I/O performance
    - **high_recovery_ops**: Optimizes recovery operations, accelerating data rebuild and repair, suitable for maintenance windows
    - **balanced**: Balanced configuration, maintaining equilibrium between client I/O and background operations
  - **Technical advantages**: Compared to traditional WRR (Weighted Round Robin) scheduler, mclock provides more precise performance isolation and QoS guarantees
- **Load balancer**: 
  - **upmap mode**: Now enabled by default in upmap mode, providing more precise PG mapping control
  - **Client requirements**: Requires setting `require_min_compat_client luminous` to ensure client supports upmap functionality
  - **Advantages**: upmap mode provides more precise data distribution balancing than traditional CRUSH reweight approach

#### Security Improvements
- **Authentication protocol**: cephx authentication protocol version 2 (`CEPHX_V2`) now required by default
- **Replay attack protection**: Added replay attack protection for authorizer, enhancing msgr v1 message signing

#### RBD Improvements
- **Fast diff mode**: Significantly improved performance in fast diff mode when comparing differences against time point
- **Cache optimization**: Shared read-only parent cache configuration option `immutable_object_cache_watermark` updated to 0.9
- **Clone functionality**: Supports cloning from non-user type snapshots
- **rbd-wnbd driver**: Gained ability for multiplexed image mapping, improving resource usage efficiency in Windows environments

#### RGW Improvements
- **Persistent Bucket notifications**: Deep integration of persistent Bucket notifications, providing more reliable notification mechanisms
- **AWS compatible API**: Added "GetTopicAttributes" API replacing existing "GetTopic" API
- **Log management**: `bilog-trim`, `datalog-trim`, `sync-error-trim` now only accept single markers for marker ranges

#### CephFS Improvements
- **🎯 Mirroring feature improvement**: CephFS mirroring functionality enhanced, evolving from Nautilus basic support to production-ready solution
  - **Improved sync mechanism**: Optimized incremental sync algorithm, greatly reducing network bandwidth usage
  - **Production ready**: Compared to Octopus technical preview version, Pacific version reaches production environment deployment standards
- **Session management**: MDS evicts clients that don't advance request tid, preventing massive session metadata accumulation
- **Disaster recovery solution optimization**: 
  - **Timed snapshots**: Incremental update mechanism, avoiding full backup overhead
  - **Smart sync**: `cephfs-mirror` daemon relies on readdir diff to identify changes in directory tree
  - **Incremental sync**: Diff applied to directories in remote filesystem, only syncing files changed between two snapshots
  - **Operational value**: Provides reliable offsite disaster recovery solution for enterprise-grade CephFS

### Important Changes

#### Terminology Changes
- **blacklist → blocklist**: All related commands and configuration options have been updated
- **Command changes**:
  ```bash
  # Old commands → New commands
  ceph osd blacklist → ceph osd blocklist
  ceph <tell|daemon> osd.<NNN> dump_blacklist → ceph <tell|daemon> osd.<NNN> dump_blocklist
  ```

#### Configuration Changes
- **scrub time configuration**: 
  - `osd_scrub_begin_hour` and `osd_scrub_end_hour` valid values are 0-23
  - `osd_scrub_begin_week_day` and `osd_scrub_end_week_day` valid values are 0-6
- **Monitor configuration**: New `mon_allow_pool_size_one` option, disabled by default
- **Pool size setting**: Setting pool size to 1 requires `--yes-i-really-mean-it` flag

#### API Changes
- **librados API**: 
  - `rados_blacklist_add` → `rados_blocklist_add`
  - `get_pool_is_selfmanaged_snaps_mode` deprecated, recommend using `pool_is_in_selfmanaged_snaps_mode`

### Important Notes
- **Upgrade path**: Must first upgrade to Octopus (15.2.z) before upgrading to Pacific
- **Client compatibility**: New clusters default to support only Luminous and newer clients
- **Authentication**: Requires cephx v2 protocol by default, old clients need configuration options
- **NFS Ganesha**: Need to delete existing nfs-ganesha clusters before upgrade, redeploy after upgrade
- **Terminology changes**: All blacklist related terminology changed to blocklist
- **Security upgrade**: Recommend upgrading as soon as possible for latest security fixes

---

## Quincy

### Release Status
- **Current version**: v17.2.9
- **Version series**: v17.2.0 to v17.2.9
- **Lifecycle**: Reached end of life (EOL), v17.2.9 is the last fix version, archived

### Main Highlights

#### Dashboard
- **Interface improvements**: Dashboard interface improvements providing better user experience
- **Feature enhancements**: More management functionality and monitoring capabilities

#### RADOS
- **🎯 mclock scheduler maturity**: mclock scheduler becomes default option, compared to Pacific version introduction, Quincy version achieves full production environment deployment
  - **Service quality guarantee**: Provides service quality priority for Ceph clients relative to background operations (such as recovery, backfill)
  - **Smart scheduling**: Dynamically allocates resources based on I/O type and priority, ensuring critical business isn't affected by background tasks
  - **Configuration simplification**: Provides preset configuration templates, reducing tuning complexity
- **Performance optimization**: Various performance improvements enhancing overall cluster performance
- **API changes**: `get_pool_is_selfmanaged_snaps_mode` C++ API deprecated, recommend using safer `pool_is_in_selfmanaged_snaps_mode` replacement

#### RBD
- **Fast diff mode**: Significantly improved performance in fast diff mode when comparing differences against time point
- **Command enhancements**: 
  - `rbd children` command added `--image-id` option, can be used for images in trash
  - Python bindings expose `RBD_IMAGE_OPTION_CLONE_FORMAT` option
  - Python bindings expose `RBD_IMAGE_OPTION_FLATTEN` option
- **Performance optimization**: Significant performance improvements for QEMU live disk sync and backup use cases

#### CephFS
- **Filesystem ID**: Can specify ID ("fscid") when creating filesystem, very useful in certain recovery scenarios
- **Recovery scenarios**: For example, monitor database loss and rebuild, recovered filesystem can have same ID as before
- **Filesystem renaming**: Supports renaming filesystem using `fs rename` command
- **MDS upgrade**: MDS upgrade no longer requires stopping all standby MDS daemons before upgrading filesystem's only active MDS
- **Failure handling**: When standby replay daemon cannot replay log, now marks rank as "damaged"

#### Other Features
- **SQLite support**: New library libcephsqlite now available
- **Virtual file system**: Provides SQLite virtual file system (VFS) on top of RADOS
- **Data striping**: Database and logs striped across multiple objects through RADOS
- **Scalability**: Almost infinitely scalable, throughput only limited by SQLite client
- **LevelDB migration**: Requires migration from LevelDB to RocksDB (if applicable)

---

## Reef

Reef is the 18th stable release of Ceph, named after the reef squid (Sepioteuthis).

### Release Status
- **Current version**: v18.2.7 (recommended for all users to upgrade)
- **Version series**: v18.2.0 to v18.2.7
- **Lifecycle**: Currently active stable version
- **⚠️ Critical fix**: v18.2.7 fixes serious BlueStore regression issues in versions 18.2.5 and 18.2.6
  - **Issue description**: BlueStore experienced data integrity issues when handling large objects (>4MB), potentially causing object read errors or corruption
  - **Impact scope**: Mainly affects RGW large file uploads, RBD large block writes, and other scenarios, may cause silent data corruption
  - **Regression cause**: BlueStore compression optimization code introduced in v18.2.5 had boundary condition handling errors
  - **Fix measures**: Rolled back problematic compression optimizations and added additional data integrity checks
  - **Upgrade advice**: If using v18.2.5 or v18.2.6, strongly recommend immediate upgrade to v18.2.7

### Main Highlights

#### Major Features
- **FileStore deprecation**: FileStore no longer supported in Reef, users need to migrate to BlueStore
- **RocksDB upgrade**: RocksDB upgraded to version 7.9.2 with significant performance improvements
- **Command updates**: `perf dump` and `perf schema` commands deprecated, replaced by new `counter dump` and `counter schema` commands
- **Cache tiering deprecation**: Cache tiering now deprecated
- **Read balancer**: New "read balancer" feature now available, allows users to balance primary PGs for each pool across cluster
- **New commands**: Added `ceph osd rm-pg-upmap-primary-all` command, allows users to clear all pg-upmap-primary mappings in osdmap

#### RGW
- **Bucket resharding**: Multi-site configurations now support bucket resharding
- **Multi-site replication**: Significant improvements in stability and consistency of multi-site replication
- **Encryption compression**: Now supports compression of objects uploaded with server-side encryption
- **Security considerations**: Before enabling `compress-encrypted` zone group feature, all zones must be upgraded to Reef

#### Dashboard
- **New pages**: New Dashboard page with improved layout
- **Card display**: Active alerts and some important charts now displayed within cards

#### RBD
- **Layered encryption**: Added support for layered client encryption
- **NBD mapping**: `try-netlink` mapping option now becomes default and deprecated, if kernel doesn't support NBD netlink interface, will retry using legacy ioctl interface

#### Telemetry
- **Leaderboard**: Users can now opt into leaderboard in telemetry public dashboard ([public dashboard](https://telemetry-public.ceph.com/))
- **Cluster description**: Users can provide cluster description that will be publicly displayed in leaderboard
- **Related commands**:
  ```bash
  # View telemetry preview
  ceph telemetry preview
  
  # Enable telemetry
  ceph telemetry on
  
  # Enable leaderboard
  ceph config set mgr mgr/telemetry/leaderboard true
  
  # Add leaderboard description
  ceph config set mgr mgr/telemetry/leaderboard_description 'Cluster description'
  ```

---

## Squid

Squid is the 19th stable release of Ceph and is the currently active stable version.

### Release Status
- **Current version**: v19.2.2 (recommended for all users to upgrade)
- **Version series**: v19.2.0 to v19.2.x
- **Lifecycle**: Currently active stable version
- **Important fixes**: v19.2.2 fixes critical RGW CopyObject data loss bug
- **Technical preview**: Crimson/Seastore technical preview version, supports RBD workloads running on replicated pools

### Main Highlights

#### Dashboard Interface Improvements
- **Redesigned navigation layout**: Navigation layout reorganized to improve usability and access to key functionality
- **CephFS snapshot and clone management**: Supports managing CephFS snapshots and clones, as well as snapshot scheduling management
- **Authorization management**: Capability to manage authorizations for CephFS resources
- **Mount helper**: Provides helper tools for CephFS volume mounting

#### Core Feature Improvements
- **CephFS subvolume labeling**: `fs subvolume create` command now supports adding labels to subvolumes through `--earmark` option
- **Extended attributes**: Extended removexattr support for CephFS virtual extended attributes
- **User accounts**: RGW user account functionality unlocks multiple AWS-compatible IAM APIs for self-service management of users, keys, groups, roles, policies, etc.

#### RADOS Improvements
- **Balancer performance**: Fixed performance bottleneck in balancer mgr module
- **HDD configuration optimization**: Based on large-scale testing, optimized default sharding configuration for HDD OSDs:
  - `osd_op_num_shards_hdd = 1` (was 5)
  - `osd_op_num_threads_per_shard_hdd = 5` (was 1)
- **BlueStore optimization**: BlueStore optimized for snapshot-intensive workloads
- **RocksDB compression**: BlueStore RocksDB LZ4 compression now enabled by default, improving average performance and "fast device" space usage
- **CRUSH rules**: New CRUSH rule type MSR (Multi-Step Retry) allows more flexible EC configurations

#### Manager Module Improvements
- **REST module**: REST manager module now trims requests based on `max_requests` option to prevent out-of-memory (OOM) issues

#### RGW Enhancements
- **Topic ownership**: SNS topics now have ownership concept, by default only owner can read/write
- **Topic policies**: Supports topic policy documents to grant permissions to other users
- **Persistent notifications**: Fixed topic parameter modification issues in persistent notifications
- **User ID**: In bucket notifications, `principalId` within `ownerIdentity` now contains complete user ID
- **Versioned buckets**: New tools to identify and fix versioned bucket index issues

#### Telemetry Improvements
- **Basic channel**: Telemetry `basic` channel now captures pool flags, helping understand feature adoption
- **Telemetry commands**: Use `ceph telemetry on` to opt into telemetry

### Important Fixes

#### Data Safety Fixes
- **RGW data loss**: Fixed critical CopyObject data loss bug when copying object to itself (fixed in v19.2.2)
- **Garbage collection**: Resolved issue where tail objects were incorrectly marked for garbage collection
- **Recovery tools**: Provided experimental `rgw-gap-list` tool to identify corrupted objects
- **Important security fixes**: Fixed critical security issues that could cause data loss

#### Performance and Stability
- **Kernel devices**: Series of optimizations for kernel device discard
- **Messaging**: Fixed connection count issues in AsyncMessenger
- **BlueStore**: Multiple BlueStore improvements and bug fixes
- **Balancer performance**: Fixed performance bottleneck in balancer mgr module
- **HDD configuration optimization**: Optimized default sharding configuration for HDD OSDs to improve mClock scheduling performance

---

## References

### Official Documentation
- [Ceph Official Documentation](https://docs.ceph.com/)
- [Ceph Release Notes](https://docs.ceph.com/en/latest/releases/)
- [Ceph Developer Documentation](https://docs.ceph.com/en/latest/dev/)

### Technical Articles
- [New in Mimic: Centralized Configuration Management](https://ceph.io/en/news/blog/2018/new-mimic-centralized-configuration-management/)
- [New in Nautilus: Crash Dump Telemetry](https://ceph.io/en/news/blog/2019/new-in-nautilus-crash-dump-telemetry/)
- [New in Octopus: Dashboard Features](https://ceph.io/en/news/blog/2020/new-in-octopus-dashboard-features/)

### Technical Talks
- Ceph Tech Talk - What's New In Octopus
- Ceph Developer Summit Quincy: RADOS Follow-up
- Diving Deep with Squid

## Special Note

This document is for reference only. If there are errors, please correct them.
