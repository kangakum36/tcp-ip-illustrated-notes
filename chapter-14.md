# Chapter 14: TCP Timeout and Retransmission

## 14.1 Introduction

### Retransmission Mechanisms

TCP has two separate mechanisms for retransmission:

1. **Timer-based retransmission**: Sets a timer on packet send and resends when timer expires (duration = RTO)
2. **Fast retransmit**: Infers losses based on:
   - Failure to advance ACK numbers (duplicate ACKs)
   - SACK information indicating out-of-order segments

*Note: This chapter focuses on detecting and retransmitting lost data. Chapter 16 covers how much data to send (congestion control).*

## 14.2 Simple Timeout and Retransmission

### Binary Exponential Backoff

- TCP uses binary exponential backoff to increase time between successive retransmissions
- Two important thresholds:
  - **R1**: Number of retries before passing "negative advice" to IP layer
  - **R2**: Number of retries before giving up on connection

### Implementation Notes

- In `tcp_timer.c`, the kernel applies binary backoff to `icsk->icsk_rto`, clamped to `TCP_RTO_MAX`
- `icsk_rto` is distinct from `srtt` and `rttvar` values (field of `inet_connection_sock`)

## 14.3 Setting the RTO (Based on RTT)

RTO is set based on:
- **SRTT**: Smoothed RTT estimate
- **rttvar**: RTT variance estimate

### 14.3.1 Classic Method

```
SRTT_i ← α(SRTT_{i-1}) + (1 - α)RTT_i
```

- EWMA of the mean, bounded by upper and lower bounds
- **Problem**: Did not work well with highly variable RTTs
- Added unnecessary retransmissions when real RTT was higher than expected

### 14.3.2 Standard Method

```
Err = M - srtt
srtt ← srtt + g(Err)
rttvar ← rttvar + h(|Err| - rttvar)
RTO = srtt + 4(rttvar)
```

- **srtt**: EWMA of the mean
- **rttvar**: EWMA of absolute error
- **TCP clock granularity**: Traditionally 500ms, more recently finer
- **RTO bounds**: Lower (1s) and upper (60s)
- **Initialization**: `srtt = M`, `rttvar = M/2` (M = first measurement)

#### Karn's Algorithm

- **Binary exponential backoff**: Backoff factor doubles when subsequent retransmission timer expires
- **Continues until**: ACK received for segment that was not retransmitted
- **ACK handling**: ACKs for retransmitted segments don't change backoff (avoids retransmission ambiguity)
- **Timestamps exception**: Allow avoiding retransmission ambiguity

#### Timestamp Operation

- **Sender**: Sends timestamp in TSV for each segment
- **Receiver**:
  - Keeps track of `TsRecent` (TSV value for next ACK)
  - Keeps track of `LastACK` (ACK number in last ACK sent)
  - Updates `TsRecent` when sequence number matches `LastACK`
  - Places `TsRecent` in TSER of every ACK
- **RTT calculation**: Sender uses TSER value and current clock

### 14.3.3 Linux Method

#### Key Differences

- **Clock granularity**: 1ms (vs traditional 500ms)
- **Uses TSOPT**: Tends to minimize rttvar due to large number of samples

#### Variables Tracked

- **srtt**: Similar to standard method
- **mdev**: Estimate of mean deviation
- **mdev_max**: Max mean deviation over last measured RTT
- **rttvar**: Always at least as large as `mdev_max`, never below 50ms

#### Constraints

- **Minimum RTO**: 200ms (due to 50ms minimum rttvar)
- **Datacenter issues**: Can be problematic for low-latency environments
- **Solutions**: Modify with `ip route` or patch kernel, or rely on fast retransmit/ECN

#### Optimizations

1. **Sample impact minimization**: `rttvar` set to max `mdev` value per RTT
2. **Lower RTT handling**: Uses lower weight (1/32 vs 1/4) for samples less than `srtt - mdev`

### 14.3.5 RTTM Robustness to Loss and Reordering

#### Out-of-Order Segments

- Generate immediate ACK for fast retransmit
- TSER set to TSV of most recent segment that advanced window
- Causes measured RTT to increase → larger RTO
- Allows more time to distinguish reordering from loss

#### Successful Retransmissions

- Use TSV value of retransmitted segment (not older segment)
- Mitigates unwanted RTO increases
- Indicates recovery is happening

## 14.4 Timer-Based Retransmission

### Timer Management

- TCP sets and cancels timers on successful transmissions
- **No data lost**: Timers never expire
- **Different from typical OS**: Optimizes for efficient setup/cancel vs setup/expiration
- **Binary backoff**: Increases on timer-based retransmission
- **Performance**: Often leads to underutilization (RTO > typical RTT)

## 14.5 Fast Retransmission

### Mechanisms

1. **Duplicate ACKs**: Required immediate ACK for out-of-order segments
2. **Loss indication**: Loss implies out-of-order delivery → duplicate ACKs

### Duplicate ACK Threshold

- **Problem**: Duplicate ACKs don't necessarily imply loss (could be reordering)
- **Solution**: `dupthresh` - number of duplicate ACKs before fast retransmit
- **Typical value**: 3 (but Linux adjusts dynamically)

### Recovery Process

- **Recovery point**: Highest sequence number sent prior to retransmission
- **Partial ACK**: ACK higher than duplicate ACK but lower than recovery point
- **Action**: Immediately resend apparently missing segment

## 14.6 Retransmission with SACK

### Advantages

- More accurately describes holes in receive window
- Faster hole filling (no additional RTT to learn about holes)

### Option Space Constraints

- **Total space**: 40 bytes for TCP options
- **TSOPT**: 10 bytes + 2 bytes padding
- **SACK overhead**: 2 bytes + 8 bytes per block
- **Typical capacity**: 3 SACK blocks

### SACK Block Ordering

1. **First block**: Most recently received segment range
2. **Other blocks**: Listed in order from previous SACK options
3. **Purpose**: Redundancy against ACK loss

### Sender Behavior

1. **Primary strategy**: Fill holes indicated by SACK blocks first
2. **Then**: Send new data if congestion control allows
3. **Advisory nature**: Cannot free segments until standard TCP ACK received

## 14.7 Spurious Timeouts and Retransmissions

### Causes

- Timeouts firing too early
- Packet reordering
- Packet duplication
- Lost ACKs

### Detection Algorithms

#### DSACK (Duplicate SACK)

- First SACK block indicates duplicate segment sequence numbers
- Can SACK already-ACKed blocks (sequence numbers below current ACK)

#### Eifel Detection

- **Method**: Stores TSV when retransmission sent
- **Detection**: If TSER < stored TSV → spurious retransmission
- **Resilience**: Works with any ACK in window covering sequence number

#### Forward Recovery (F-RTO)

- **No options required**
- **Process**: After retransmission timer expires, wait for first 2 ACKs
- **Decision**: If either is duplicate → legitimate; otherwise continue sending new data

#### Eifel Response

Standard operations after spurious retransmission detection:

```
srtt_prev = srtt + 2(G)  // G = TCP clock granularity
rttvar_prev = rttvar
```

After detection:
- **Spurious timeout**: Adjust sequence number to first unsent segment
- **Late spurious timeout**: Don't adjust (ACK already sent)

Recovery:
```
srtt ← max(srtt_prev, m)
rttvar ← max(rttvar_prev, m/2)
RTO = srtt + max(G, 4(rttvar))
```

## 14.8 Packet Reordering and Duplication

### Reordering Challenges

- **IP characteristic**: No guarantee of packet ordering
- **Paths**: Can occur in forward or reverse direction
- **Reverse path**: Causes burstiness in sending pattern
- **Forward path**: Hard to distinguish from loss → spurious fast retransmits

### Reordering Solutions

- **dupthresh**: Helps with mild reordering
- **Linux**: Adjusts dupthresh dynamically
- **Modern approach**: RACK algorithms (see `tcp_recovery` documentation)

### Duplication

- **Cause**: IP can deliver packets multiple times
- **Effect**: ACKs from duplicates can trigger spurious fast retransmits
- **Solution**: DSACK can suppress spurious retransmissions

## 14.9 Destination Metrics

### Learning and Persistence

- **Runtime learning**: srtt, rttvar, packet reordering amounts
- **Historical approach**: Lost when connection closes
- **Modern approach**: Maintain in routing/forwarding table
- **Benefit**: Initialize new connections with recent experience

## 14.10 Repacketization

### Capability

- **Allowed**: TCP can send bigger segments on retransmission
- **Constraints**: Must stay under receiver MSS and path MTU
- **Benefit**: Improved performance through fewer segments

### Implementation Detail

- **Byte-oriented**: TCP identifies data by byte numbers, not segment numbers
- **Flexibility**: Retransmitted segment doesn't need to match original exactly

## Miscellaneous Notes

### Kernel Timer Implementation

- **timer_base**: Per-CPU timers list
- **timer_list**: Linked list of timers
- **CPU mobility**: Timers can be moved between CPUs
- **Scheduling**: Reacts to timer expirations via interrupts