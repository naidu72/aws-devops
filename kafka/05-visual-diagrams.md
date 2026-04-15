# Visual Diagrams - AWS MSK Interview Prep

## 1. TOPIC CREATION COMMAND BREAKDOWN

```
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --command-config $MSK_CONFIG \
                --create --topic msk-pprd \
                --partitions 3 \
                --replication-factor 2

Command Parser:
  bootstrap-server  → Which brokers? → $BOOTSTRAP (both)
  command-config    → Auth config file → $MSK_CONFIG (credentials)
  create            → Action → Create new topic
  topic             → Name → msk-pprd (your topic)
  partitions        → Count → 3 (divide into 3 queues)
  replication-factor → Copies → 2 (2 copies of each)

Result:
  ✓ Topic created
  ✓ 3 partitions spread across 2 brokers
  ✓ Each partition replicated on 2 brokers
```

---

## 2. INITIAL PARTITION ASSIGNMENT

```
After creation, Kafka assigns partitions:

Option A: Balanced Distribution
┌────────────────────────────────────────┐
│          KAFKA CLUSTER                 │
├────────────────────────────────────────┤
│                                        │
│  Broker 0            Broker 1          │
│  ┌──────────────┐   ┌──────────────┐  │
│  │ P0 (Leader)  │   │ P0 (Follower)│  │
│  │ P2 (Leader)  │   │ P1 (Leader)  │  │
│  │ P1 (Follower)│   │ P2 (Follower)│  │
│  └──────────────┘   └──────────────┘  │
│                                        │
│  Total: P0,P1,P2   Total: P0,P1,P2   │
│         3 leaders        1 leader     │
│                                        │
└────────────────────────────────────────┘

Partition Details:
- P0: Leader=Broker 0, Replicas=[0,1], ISR=[0,1] ✓
- P1: Leader=Broker 1, Replicas=[1,0], ISR=[1,0] ✓
- P2: Leader=Broker 0, Replicas=[0,1], ISR=[0,1] ✓
```

---

## 3. MESSAGE FLOW (NORMAL STATE)

```
Producer sends message to topic msk-pprd:

Step 1: Producer creates message
┌─────────────────────┐
│  Message: "Hello"   │
│  Key: "user123"     │
│  Value: "data"      │
└─────────────────────┘

Step 2: Partition selection
Key "user123" → Hash → Partition 2
(Uses partitioner to decide which partition)

Step 3: Send to leader of P2
┌─────────────────────────────────────────────────┐
│ Producer → Broker 0 (Leader of P2)              │
│            "Write message to P2"                │
└─────────────────────────────────────────────────┘

Step 4: Leader appends to partition
Broker 0 Partition 2:
  [Message 1, Message 2, Message 3, ... Message N]
                                            ↑ New

Step 5: Replicate to follower
Broker 0 → Broker 1 (Follower of P2)
"Replicate message to P2"

Broker 1 Partition 2:
  [Message 1, Message 2, Message 3, ... Message N]
                                            ↑ New (copy)

Step 6: Return ack to producer
Broker 0 → Producer: "OK, written safely"
(If acks=all, waits for Broker 1 first)
```

---

## 4. BROKER FAILURE - BROKER 0 DIES

```
BEFORE (Normal State):
┌──────────────────────────────────────────────┐
│ Broker 0 (UP) ✓        Broker 1 (UP) ✓      │
├──────────────────────────────────────────────┤
│ P0 [L]  ─→ Replicates to ─→ P0 [F]          │
│ P1 [F]  ←─ Replicates from ←─ P1 [L]        │
│ P2 [L]  ─→ Replicates to ─→ P2 [F]          │
│                                              │
│ Producers can write to all partitions ✓     │
│ Consumers can read from all partitions ✓    │
└──────────────────────────────────────────────┘

FAILURE: Broker 0 dies
         Network cable cut
         Disk failure
         Memory exhaustion

┌──────────────────────────────────────────────┐
│ Broker 0 (DOWN) ✗       Broker 1 (UP) ✓     │
├──────────────────────────────────────────────┤
│ ✗ Unreachable           P0 [L]  (promoted!) │
│                         P1 [L]  (still)     │
│                         P2 [L]  (promoted!) │
│                                              │
│ Producers: Send to Broker 1 instead ✓       │
│            (automatic failover)             │
│ Consumers: Read from Broker 1 ✓             │
│            (have replicas)                  │
└──────────────────────────────────────────────┘

Timeline:
0ms   - Broker 0 dies
100ms - Broker detects dead (via heartbeat)
500ms - Leader election starts
2000ms- Failover complete
5000ms- Producers/Consumers reconnect
10s   - Back to normal

Data Safety: ✓ NO DATA LOSS (all on Broker 1 as follower)
Availability: ✓ STILL READING/WRITING
Risk Level: ⚠️ HIGH (no fault tolerance left, RF=1 now)
```

---

## 5. BOTH BROKERS DIE (DISASTER)

```
Normal:
┌────────────────────┐
│ B0 [UP] ✓ │ B1 [UP] ✓
│ P0,P1,P2 │ P0,P1,P2
└────────────────────┘

After Broker 0 dies:
┌────────────────────┐
│ B0 [DOWN]✗│ B1 [UP] ✓
│          │ P0,P1,P2 (all leader now)
└────────────────────┘
Status: Degraded but working

If Broker 1 also dies:
┌────────────────────┐
│ B0 [DOWN]✗│ B1 [DOWN]✗
│          │
│ ALL GONE! DISASTER!
└────────────────────┘

Consequences:
✗ Topic completely unavailable
✗ All data lost (both copies gone)
✗ No recovery (from Kafka perspective)
✗ Application crashes

Recovery: Only from backups/external storage
Timeline to total loss: 2-5 minutes
Probability: Low but catastrophic

How to prevent:
- Add 3rd broker (RF=3)
- Daily snapshots
- Backup to external storage (S3)
```

---

## 6. PRODUCER LATENCY vs DURABILITY TRADE-OFF

```
acks=0 (Fastest, Least Safe)
───────────────────────────
Producer              Broker 0
Send message ────────→ (done)
Return "OK" ←────────

Timeline: ~5ms
Data in broker? Unknown
If broker dies: Message lost
Use: Metrics, non-critical logs

⚡ Latency: Fastest
✗ Durability: Worst


acks=1 (Medium)
───────────────────────────
Producer              Broker 0
Send message ────────→ Store
              ←────── "OK"
Return "OK" to app

Timeline: ~10-20ms
Data in broker? Yes, on leader
If leader dies before replicate: Lost
Use: Most production systems

⚡⚡ Latency: Medium
✓ Durability: Good


acks=all (Slowest, Most Safe)
───────────────────────────
Producer   Broker 0      Broker 1
Send ────────→ Store ────────→ Store
       ←────────────────────← "OK"
        ←─────────────────
Return "OK"

Timeline: ~50-100ms
Data in broker? Yes, on all ISR
If any broker dies: Still safe
Use: Financial transactions, critical

🐌 Latency: Slowest
✓✓ Durability: Excellent


Trade-off Graph:
Durability  ^
    │       │ acks=all (safe but slow)
    │       │          ╱
    │       │       ╱
    │       │   ╱
    │       │ ╱ acks=1 (balanced)
    │   ╱╱╱
    │ ╱ acks=0 (fast but risky)
    └──────────────────────────► Latency
```

---

## 7. CONSUMER GROUP PARTITION ASSIGNMENT

```
Your Setup: 3 Partitions, 2 Brokers
Consumer Group: "payment-processor"

Case 1: 1 Consumer
┌────────────────────────────┐
│ Consumer 1                 │
│ ├─ Partition 0             │
│ ├─ Partition 1             │
│ └─ Partition 2             │
│ Reads all 3 sequentially   │
└────────────────────────────┘

Throughput: ~1000 msg/sec
Usage: All CPU on 1 machine
Parallelism: None (sequential)


Case 2: 2 Consumers
┌──────────────────┬──────────────────┐
│ Consumer 1       │ Consumer 2       │
│ ├─ Partition 0   │ ├─ Partition 1   │
│ └─ Partition 2   │                  │
│ (uneven)         │                  │
└──────────────────┴──────────────────┘

Throughput: ~2000 msg/sec (parallel)
Usage: CPU on 2 machines
Parallelism: Partial


Case 3: 3 Consumers (Optimal)
┌──────────┬──────────┬──────────┐
│ Consumer1│ Consumer2│ Consumer3│
│ Part 0   │ Part 1   │ Part 2   │
└──────────┴──────────┴──────────┘

Throughput: ~3000 msg/sec (max for 3 partitions)
Usage: CPU on 3 machines
Parallelism: Full ✓


Case 4: 4 Consumers (Wasted)
┌──────────┬──────────┬──────────┬──────────┐
│ Consumer1│ Consumer2│ Consumer3│ Consumer4│
│ Part 0   │ Part 1   │ Part 2   │ [IDLE]   │
└──────────┴──────────┴──────────┴──────────┘

Throughput: ~3000 msg/sec (same as 3)
Usage: CPU on 3 machines (1 sits idle)
Parallelism: Wasted resource

Rule: Consumers ≤ Partitions
```

---

## 8. OFFSET MANAGEMENT

```
Partition 0 Timeline:
[1][2][3][4][5][6][7][8][9][10]
↑                           ↑
First message          Latest message

Consumer Group Reading:
        Committed Offset (last processed)
                ↓
[1][2][3][4][5][6][7][8][9][10]
                      ↑
                  Consumer position
                  (next to read)

Lag = Latest - Committed
    = 10 - 5 = 5 messages behind

Consumer poll():
Gets [6][7][8][9][10]
Processes them
Commits offset to 10

Next poll():
Lag = 10 - 10 = 0 ✓ (caught up!)


Scenario: Consumer crashes after processing, before commit
[1][2][3][4][5][6][7][8][9][10]
              ↑                 ↑
          Committed         Processed
          (offset 5)        (reached 9)

Restart:
Starts from offset 5 (committed)
Will reprocess [6][7][8][9]
Duplicates, but acceptable
```

---

## 9. ISR (IN-SYNC REPLICAS) VISUALIZATION

```
Normal State (All replicas healthy):
Partition 0:
  Leader: Broker 0
  Replicas: [0, 1]
  ISR: [0, 1] ✓ (all in sync)

Broker 1 Network Latency (lagging):
  Leader: Broker 0
  Replicas: [0, 1]
  ISR: [0] ⚠️ (follower lagging, removed from ISR)
  Follower replicas: [1] (catching up in background)

After catch-up:
  ISR: [0, 1] ✓ (back in sync)

Broker 1 Fully Down:
  Leader: Broker 0
  Replicas: [0, 1]
  ISR: [0] (broker 1 gone, removed)
  Available replicas: [0] (only leader)

If Broker 0 dies now:
  ISR was [0]
  Broker 0 (leader) gone
  No replica to promote!
  → Data lost if no backups


Why ISR matters:
- min.isr setting: Writes require ISR size
- If current ISR < min.isr → Writes fail
- Safety guarantee: Can lose up to (RF - min.isr) brokers
- Your setup: RF=2, min.isr should be 2
  → Can lose 0 brokers without write failure
  → If 1 broker fails, ISR=[1], writes fail (min.isr=2)
  → This is SAFE (prevents data corruption)
```

---

## 10. FAILURE RECOVERY TIMELINE

```
Broker 0 Dies at T=0:

T=0ms: Broker 0 dies (hardware failure)
       Broker 1 still running
       Producers try to connect to B0 → Timeout

T=500ms: Broker 1 controller detects B0 dead
         Starts leader election for B0's partitions

T=1000ms: Leader election complete
          New leaders selected from ISR
          B1 takes over all partitions
          Partition metadata updated

T=2000ms: Producers get metadata update
          Learn about new leaders on B1
          Reconnect and continue writing ✓

T=5000ms: Consumers discover new leaders
          Get metadata update
          Continue reading ✓

T=10000ms: System fully operational again
           All traffic on B1
           No data loss (RF=2 guaranteed copies)
           But fragile (RF=1 now) ⚠️

T=3600s (1 hour): B0 comes back online
                  Starts replicating data from B1
                  Catches up on backlog
                  Becomes follower again

T=3700s: ISR = [0, 1] again ✓
         Full fault tolerance restored
         Can lose 1 broker again safely
```

---

## 11. PRODUCER vs CONSUMER PERSPECTIVE (SAME TOPIC)

```
Single Topic msk-pprd with 3 partitions:

PRODUCER PERSPECTIVE:
┌─────────────────┐
│  Send Messages  │
└────────┬────────┘
         │ (routing based on key)
         ↓
    ┌────┴────┬──────────┬──────────┐
    ↓         ↓          ↓          
 Part 0    Part 1     Part 2     [Brokers]
 [1,4,7]   [2,5,8]    [3,6,9]
    ↑         ↑          ↑
    └────┬────┴──────────┘
         ↓
Stored on 2 brokers
(RF=2 copies)


CONSUMER PERSPECTIVE (Same Topic):
Group A ("analytics"):
┌──────────────────────────────────┐
│ Consumer A1 ─→ Partition 0 [1,4,7]
│ Consumer A2 ─→ Partition 1 [2,5,8]
│ Consumer A3 ─→ Partition 2 [3,6,9]
└──────────────────────────────────┘

Group B ("fraud-detection"):
┌──────────────────────────────────┐
│ Consumer B1 ─→ All partitions    │
│              [1,2,3,4,5,6,7,8,9]│
└──────────────────────────────────┘

Same data, different consumers!
Group A: Parallel processing (3 consumers)
Group B: Sequential processing (1 consumer)
No interference between groups
```

