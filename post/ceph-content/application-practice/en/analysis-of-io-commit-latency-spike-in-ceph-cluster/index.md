# Analysis of IO Commit Latency Spike in Ceph Cluster

## Symptom
- **Environment**: After abnormal node reboot in Ceph cluster
- **Affected Metric**: Prometheus rate value of `ceph_osd_op_w_latency`
- **Behavior**:
  - Pre-reboot: Values showed normal increment (peak ~1M)
  - Post-reboot: 
    - Started recording from 0
    - Spiked to 4.2B after 3 minutes (close to 2³²)
    - ![1cbdc1281f3f34a96a1cf40b55d69e44.jpg|600](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/1cbdc1281f3f34a96a1cf40b55d69e44.jpg)

## Investigation Process
### Phase 1: Initial Hypotheses
| Hypothesis               | Verification Method          | Conclusion                      |
|--------------------------|------------------------------|---------------------------------|
| Prometheus calculation   | Reviewed rate() function     | Confirmed proper counter reset  |
| Ceph stat initialization | Inspected OSD.cc init code   | Verified proper atomic init     |

### Phase 2: Deep Analysis
1. **Key Findings**:
   - 3-minute zero period before spike
   - 4.2B spike caused axis display anomaly

2. **Mechanism Analysis**:
   - Uses system realtime clock for time delta
   - No negative value protection

### Phase 3: Verification
1. **Environmental Evidence**:
   - NTP logs showed 4ms time reversal
   - ![image.png|600](https://raw.githubusercontent.com/YLShiJustFly/picturebed/main/images/20250608172327.png)

2. **Reproduction Method**:
   - Applied IO load via fio
   - Manually rewound system time by 1ms
   - Successfully reproduced the spike
   - ![](https://popofp.vipfp.ps.netease.com/file/684162b6d5dbab9ee1d1d45251JL0trO01)

## Root Cause
1. **Direct Cause**:
   - System time reversal caused negative time delta
   - Unsigned integer conversion

2. **Underlying Cause**:
   - Incomplete adoption of monotonic clock
   - Related PR #39273 <https://github.com/ceph/ceph/pull/39273> didn't replace all realtime usages

## Solution
1. **Long-term Improvement**:
   - Promote complete migration to monotonic clock
   - Or add time delta validation logic
2. Reported issue on official Slack <https://tracker.ceph.com/issues/71580>, currently being addressed by maintainers <https://github.com/ceph/ceph/pull/64003>
