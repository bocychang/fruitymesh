# Active Queue Management (AQM) Implementation

## Overview

This document describes the implementation of Active Queue Management (AQM) based on RED (Random Early Detection) variant to improve FruityMesh network performance under congestion.

## Problem Statement

When LOW priority queues become saturated, they can interfere with HIGH priority packet delivery rate (PDR), causing network congestion and potential deadlock scenarios. This impacts overall mesh network reliability and throughput.

## Solution Components

### 1. RED-Variant Active Queue Management

The implementation uses a RED (Random Early Detection) variant algorithm that proactively drops LOW and MEDIUM priority packets when queue depth exceeds thresholds:

#### Algorithm Details:
- **LOW Priority Thresholds**:
  - Minimum: `AQM_LOW_QUEUE_MIN_THRESHOLD = 15` packets (start dropping)
  - Maximum: `AQM_LOW_QUEUE_THRESHOLD = 25` packets (max drop probability)
  - Max Drop Probability: `AQM_MAX_DROP_PROBABILITY = 70%`
  
- **MEDIUM Priority Thresholds** (more lenient than LOW):
  - Minimum: `AQM_MEDIUM_QUEUE_MIN_THRESHOLD = 20` packets
  - Maximum: `AQM_MEDIUM_QUEUE_THRESHOLD = 30` packets
  - Max Drop Probability: `AQM_MEDIUM_MAX_DROP_PROBABILITY = 40%`

- **Adaptive Aggressiveness**:
  - When HIGH queue depth > 10 packets (under pressure):
    - LOW: min threshold → 10, max drop → 85%
    - MEDIUM: min threshold → 15, max drop → 60%
  - Severe congestion (queue >= 150% of threshold): deterministic drop

- **Drop Probability Calculation**:
  - Linear ramp from 0% to max between MIN and MAX thresholds
  - Randomized dropping to avoid global synchronization
  - Formula: `dropProb = (queueDepth - MIN) * MAX_DROP / (MAX - MIN)`

#### Application:
- Applied only to LOW and MEDIUM priority packets
- VITAL and HIGH priority packets are never dropped by AQM
- Prevents queue congestion proactively rather than reactively

### 2. Priority-Based Retransmission Policy

The implementation introduces differentiated reliability based on message priority:

#### Transmission Modes:
- **HIGH & VITAL Priority**: 
  - Use reliable transmission (WRITE_REQ)
  - BLE stack provides automatic retransmission
  - Guaranteed delivery (best effort)
  
- **LOW & MEDIUM Priority**:
  - Use unreliable transmission (WRITE_CMD)
  - No retransmission
  - Lower overhead, faster transmission
  - Acceptable packet loss for non-critical data

#### Benefits:
- Reduces network congestion from retransmissions
- Gives HIGH priority packets better delivery guarantees
- Improves overall network throughput
- Reduces latency for critical messages

## Configuration Parameters

All parameters can be configured in `Config.h`:

```cpp
// LOW Priority Queue Thresholds (most aggressive)
#define AQM_LOW_QUEUE_THRESHOLD 25              // Start max drop at 25 packets
#define AQM_LOW_QUEUE_MIN_THRESHOLD 15          // Start dropping at 15 packets
#define AQM_MAX_DROP_PROBABILITY 70             // Up to 70% drop rate

// MEDIUM Priority Queue Thresholds (moderate)
#define AQM_MEDIUM_QUEUE_THRESHOLD 30           // Start max drop at 30 packets
#define AQM_MEDIUM_QUEUE_MIN_THRESHOLD 20       // Start dropping at 20 packets
#define AQM_MEDIUM_MAX_DROP_PROBABILITY 40      // Up to 40% drop rate

// Enable/disable priority-based retransmission policy
#define ENABLE_PRIORITY_BASED_RETRANSMISSION 1
```

### Tuning Guidelines:

**If HIGH priority PDR is still low (<90%)**:
- Decrease `AQM_LOW_QUEUE_MIN_THRESHOLD` (more aggressive, e.g., 10)
- Increase `AQM_MAX_DROP_PROBABILITY` (e.g., 80-90%)
- Decrease thresholds for MEDIUM as well

**If LOW priority PDR is too low (<50%)**:
- Increase `AQM_LOW_QUEUE_THRESHOLD` (less aggressive, e.g., 30)
- Decrease `AQM_MAX_DROP_PROBABILITY` (e.g., 50-60%)

**For balanced performance**:
- Current settings are optimized for HIGH priority protection
- Monitor statistics to adjust based on traffic patterns

## Statistics and Monitoring

### Available Metrics:

1. **Per-Connection Statistics** (via `BaseConnection`):
   - `GetAQMDroppedPacketsLow()`: Count of LOW priority packets dropped by AQM
   - `GetAQMDroppedPacketsMedium()`: Count of MEDIUM priority packets dropped by AQM
   - `GetAQMTotalAttempts()`: Total queue attempts for drop rate calculation

2. **Queue Statistics** (via `ChunkedPriorityPacketQueue`):
   - `aqmDroppedPacketsLow`: Cumulative LOW priority drops
   - `aqmDroppedPacketsMedium`: Cumulative MEDIUM priority drops
   - `aqmTotalAttempts`: Total enqueue attempts

### Logging:

AQM events are logged with the "AQM" tag:
```
logt("AQM", "Dropped LOW priority packet due to queue congestion (queue depth: %u)", queueDepth);
```

Priority-based retransmission changes are logged with the "CONN" tag:
```
logt("CONN", "Downgraded LOW priority packet to unreliable transmission");
```

## Implementation Details

### Files Modified:

1. **`Config.h`**: Added AQM configuration constants
2. **`ChunkedPriorityPacketQueue.h/.cpp`**: 
   - Implemented `ShouldDropPacketAQM()` function
   - Modified `SplitAndAddMessage()` to check AQM before queueing
   - Added statistics tracking
3. **`BaseConnection.h/.cpp`**:
   - Modified `QueueData()` to adjust delivery option based on priority
   - Added AQM statistics accessors
4. **`ChunkedPacketQueue.h/.cpp`**: Existing queue implementation (no changes needed)

### Key Functions:

#### `ChunkedPriorityPacketQueue::ShouldDropPacketAQM()`
```cpp
bool ChunkedPriorityPacketQueue::ShouldDropPacketAQM(DeliveryPriority prio) const
{
    // Only apply to LOW and MEDIUM priority
    if (prio == DeliveryPriority::VITAL || prio == DeliveryPriority::HIGH)
        return false;
    
    const u32 queueDepth = queues[(u32)prio].GetAmountOfPackets();
    
    // Below minimum threshold: no dropping
    if (queueDepth < AQM_LOW_QUEUE_MIN_THRESHOLD)
        return false;
    
    // Calculate drop probability and make random decision
    // ...
}
```

#### `BaseConnection::QueueData()` - Priority-Based Retransmission
```cpp
#if ENABLE_PRIORITY_BASED_RETRANSMISSION == 1
    if (messagePriority == DeliveryPriority::LOW || 
        messagePriority == DeliveryPriority::MEDIUM)
    {
        // Force unreliable transmission (no retransmission)
        if (adjustedDeliveryOption == DeliveryOption::WRITE_REQ)
        {
            adjustedDeliveryOption = DeliveryOption::WRITE_CMD;
        }
    }
#endif
```

## Performance Impact

### Expected Improvements:

1. **Reduced Congestion**:
   - Proactive packet dropping prevents queue overflow
   - Lower buffer utilization under high load
   - Prevents deadlock scenarios

2. **Better HIGH Priority PDR**:
   - HIGH priority packets experience less queuing delay
   - Reliable transmission ensures delivery
   - Less interference from LOW priority traffic

3. **Overall Throughput**:
   - Reduced retransmission overhead
   - More efficient buffer utilization
   - Better fairness between priority classes

### Trade-offs:

1. **Packet Loss**:
   - LOW/MEDIUM priority packets may be dropped
   - Acceptable for non-critical data
   - Applications should handle packet loss gracefully

2. **Configuration Sensitivity**:
   - Thresholds may need tuning for specific scenarios
   - Drop probability affects aggressiveness
   - Monitor statistics to optimize parameters

## Testing and Validation

### Recommended Tests:

1. **Congestion Scenarios**:
   - Generate high volume of LOW priority traffic
   - Verify HIGH priority packets maintain high PDR
   - Monitor queue depths and drop statistics

2. **Priority Mix**:
   - Mix HIGH and LOW priority traffic
   - Verify correct prioritization
   - Check retransmission behavior

3. **Threshold Tuning**:
   - Test different AQM thresholds
   - Measure impact on throughput and latency
   - Optimize for target scenario

### Metrics to Monitor:

- Queue depth over time
- Packet drop rates by priority
- End-to-end delivery success rates
- Network throughput
- Latency distribution

## Future Enhancements

1. **Adaptive Thresholds**:
   - Dynamically adjust based on network conditions
   - Learning algorithms for optimal parameters

2. **Per-Connection AQM**:
   - Different thresholds per connection type
   - Connection-specific congestion control

3. **Enhanced Statistics**:
   - Time-series data collection
   - Drop rate trending
   - Performance analytics

4. **Alternative Algorithms**:
   - PIE (Proportional Integral controller Enhanced)
   - CoDel (Controlled Delay)
   - FQ-CoDel (Fair Queueing with CoDel)

## References

- RED (Random Early Detection): Floyd, S., & Jacobson, V. (1993)
- FruityMesh Quality of Service documentation
- BLE GATT reliability mechanisms

## Contact

For questions or issues related to AQM implementation, please create an issue in the FruityMesh repository.
