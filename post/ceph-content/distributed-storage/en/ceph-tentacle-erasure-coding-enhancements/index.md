
## Vision and Objectives

**Core Goal**: Optimize I/O performance for Erasure Coded pools to be similar to Replicated Pools

**Primary Objectives**:
- Lower Total Cost of Ownership (TCO)
- Make Erasure Coded pools viable for use with block and file storage

## Enabling "Optimised" EC

### Important Considerations
- **Default State**: All optimizations are turned off by default
- **Per-Pool Configuration**: Optimizations can be enabled for each pool individually
- **⚠️ Irreversible Operation**: OPTIMIZATIONS CANNOT BE SWITCHED OFF once enabled
- **Version Requirements**: All OSDs, MONs, and MGRs must be upgraded to Tentacle or later
- **Backward Compatibility**: Compatible with old clients

### Configuration Methods

#### Enable optimizations for a specific pool
```bash
ceph osd pool set <pool_name> allow_ec_optimizations true
```

#### Enable optimizations by default for new pools
```ini
[mon]
osd_pool_default_flag_ec_optimizations = true
```

## Key Technical Features

### Previously Implemented Core Features
- **Partial Reads**
- **Partial Writes**
  - Note: Partial metadata – unwritten shards have no processing
- **Parity Delta Writes**
  - Per-IO auto-switch between write methods
- **Larger Default Chunk Size**
- **Direct Read**
- **Direct Write**

### New Important Features

#### 1. Space Efficient Small Objects

**Scenario**: 16k Chunk Size, 4+2 EC configuration

| Method | 4k Object Space Consumption | Space Efficiency |
|--------|----------------------------|------------------|
| Legacy | 96k | Baseline |
| Optimized | 12k | **8x Improvement** |

#### 2. Space Efficient Sparse Writes

**Scenario**: 16k Chunk Size, 4+2 EC, Object >64k, write to offset 40k

| Method | 4k Write Space Consumption | Space Efficiency |
|--------|---------------------------|------------------|
| Legacy | 96k | Baseline |
| Optimized | 12k | **8x Improvement** |

**Importance**: This is very important for RBD (RADOS Block Device)!

#### 3. Re-balance Improvements

**Legacy Issues**:
- Required reading all data shards
- Recovery operations included log recovery and re-balance

**Optimized Improvements**:
- Read from old OSD only
- Write operations remain optimal
- Significantly improved re-balance efficiency

#### 4. Deep Scrub Enhancements

**Technical Approach**:
- Using properties of CRC mathematics and common EC techniques
- Calculate coding CRC from data CRC
- Single corrupt shard identification (for m > 1)
- Shard CRCs do not need to be stored in metadata
- New metadata checks for "partial" shards

**Credits**: Thanks to Jon Bailey!

## Performance Analysis

### Read Performance
- **Random Read Performance**: 3-4x improvement in 4k random read performance
- **Comparison**: Still not matching 3-way replica, but compelling

### 4k Workload Performance Analysis
- **Chunk Size**: Recommended to set to 16k
- **k Value Impact**: Value of k has minimal impact on performance
- **Latency**: Still higher than replica
- **Limiting Factors**:
  - Lack of write cache means cannot hide latency of Read-Modify-Write
  - Direct-reads may improve latency and throughput

### Random R/W Performance (70:30 ratio)
- **Universal Improvement**: All workloads show performance improvement
- **Large IO Advantage**: Very large IO (Object) workloads show improvement, though with smaller increases

## Configuration Recommendations

### Best Practices
- **Recommended Chunk Size**: 16k
- **k Value Selection**: Minimal impact on performance, choose based on fault tolerance requirements
- **Latency Optimization**: Consider enabling direct read functionality
- **Use Cases**: Particularly suitable for RBD and scenarios requiring high space efficiency

### Performance Trade-offs
- Read-Modify-Write operation latency still exists
- Slightly higher latency compared to 3-way replica, but significantly improved space efficiency
- Overall performance improvement is significant, making EC pools viable in more scenarios

## Summary

The Ceph Tentacle Erasure Coding enhancements address the following key areas:

1. **Space Efficiency**: Eliminates space waste in small object and sparse write scenarios
2. **Performance Enhancement**: Significantly improves overall I/O performance
3. **Operational Efficiency**: Improves re-balance and deep scrub mechanisms
4. **Expanded Use Cases**: Makes EC pools more viable for block and file storage scenarios

These improvements make Erasure Coded pools a compelling storage solution that achieves an excellent balance between cost and performance, particularly for scenarios where storage efficiency is paramount while maintaining acceptable performance characteristics.

## References

1. Reddit Community Discussion. (2025). *Massive EC improvements with Tentacle release*. r/ceph. Retrieved from https://www.reddit.com/r/ceph/comments/1lj1fns/massive_ec_improvements_with_tentacle_release/

2. Scales, B., & Ainscow, A. (2025). *Erasure Coding Enhancements for Tentacle*. Ceph Day London 2025. Retrieved from https://ceph.io/assets/pdfs/events/2025/ceph-day-london/04%20Erasure%20Coding%20Enhancements%20for%20Tentacle.pdf

3. Ceph Development Team. (2025). *Ceph London 2024: Erasure Coding Optimizations*. [Video presentation]. https://youtu.be/bwcntmYP7ic

4. Ceph Documentation. (2025). *Ceph Storage Cluster Configuration Reference*. Retrieved from https://docs.ceph.com/

5. Bailey, J. (Contributing author). Deep Scrub enhancement algorithms and CRC mathematics implementation for Ceph Tentacle EC optimizations.

---

*Note: This document was compiled for the ceph-deep-dive GitHub knowledge repository, based on community discussions and official Ceph documentation.*
