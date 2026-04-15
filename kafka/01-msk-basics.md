# AWS MSK & Kafka Interview Prep - Basics

## Your Setup
- **Brokers**: 2 brokers in AWS MSK cluster
- **Topic**: `msk-pprd`
- **Partitions**: 3
- **Replication Factor**: 2

---

## 1. VISUAL: Topic Distribution Across Brokers

```
Your MSK Cluster (2 Brokers):
┌─────────────────────────────────────────────────────┐
│                    CLUSTER                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Broker 1                          Broker 2        │
│  ┌──────────────┐                 ┌──────────────┐ │
│  │ PARTITION 0  │                 │ PARTITION 1  │ │
│  │ (Leader)     │                 │ (Leader)     │ │
│  │ Replicas:[0] │                 │ Replicas:[1] │ │
│  └──────────────┘                 └──────────────┘ │
│                                                     │
│  │                 │ PARTITION 2   │               │
│  │                 │ (Leader)      │               │
│  │                 │ Replicas:[0,1]│               │
│  │                 │ (split between)│              │
│  │                 │                │              │
│  └──────────────────────────────────────────────────┘
│
└─────────────────────────────────────────────────────┘

Replication Strategy: Each partition has 2 replicas
- Partition 0: Leader on Broker 0, Follower on Broker 1
- Partition 1: Leader on Broker 1, Follower on Broker 0
- Partition 2: Both Broker 0 & Broker 1
```

---

## 2. SIMPLE PARTITION ASSIGNMENT TABLE

| Partition | Leader Broker | Replica Brokers | In-Sync Replicas |
|-----------|---------------|-----------------|------------------|
| 0         | Broker 0      | [0, 1]          | [0, 1]           |
| 1         | Broker 1      | [1, 0]          | [1, 0]           |
| 2         | Broker 0      | [0, 1]          | [0, 1]           |

---

## 3. KEY CONCEPTS - SIMPLE EXPLANATIONS

### Partition
- **What**: Divide topic into chunks
- **Why**: Parallelism & scale horizontally
- **Your setup**: 3 partitions = 3 separate queues

### Replication Factor = 2
- **What**: Each partition copied to 2 brokers
- **Why**: Fault tolerance & high availability
- **Example**: If Broker 0 dies, Broker 1 still has the data

### Leader vs Follower
- **Leader**: Handles all read/write requests
- **Follower**: Just copies data (standby)
- **If Leader dies**: Broker 1 becomes new leader

### In-Sync Replicas (ISR)
- **Definition**: Replicas that are caught up with leader
- **Importance**: Only ISR can become new leader
- **Your setup**: Normally all 2 replicas are in ISR

---

## 4. SCENARIO: WHAT HAPPENS IF BROKER 0 DIES?

```
BEFORE (Normal):
┌─────────────────────────────────────┐
│ Broker 0        │      Broker 1     │
│ - P0 (Leader)   │  - P1 (Leader)    │
│ - P2 (Leader)   │  - P2 (Follower)  │
└─────────────────────────────────────┘

AFTER (Broker 0 Dies):
┌─────────────────────────────────────┐
│   BROKER 0 DOWN   │      Broker 1    │
│       ✗           │ - P0 (Leader)    │
│                   │ - P1 (Leader)    │
│                   │ - P2 (Leader)    │
└─────────────────────────────────────┘

Result:
✓ All partitions still readable/writable
✓ All 3 partitions now on Broker 1
✓ Replication factor drops to 1 temporarily
✗ Data only on 1 broker (risky!)
✗ No fault tolerance until Broker 0 comes back
```

---

## 5. SCENARIO: WHAT IF BROKER 1 DIES?

```
BEFORE:
┌─────────────────────────────────────┐
│ Broker 0        │      Broker 1     │
│ - P0 (Leader)   │  - P0 (Follower)  │
│ - P2 (Leader)   │  - P1 (Leader)    │
│                 │  - P2 (Follower)  │
└─────────────────────────────────────┘

AFTER (Broker 1 Dies):
┌─────────────────────────────────────┐
│ Broker 0        │   BROKER 1 DOWN   │
│ - P0 (Leader)   │        ✗          │
│ - P1 (Leader)   │                   │
│ - P2 (Leader)   │                   │
└─────────────────────────────────────┘

Result:
✓ All partitions still leader on Broker 0
✓ All data accessible from Broker 0
✗ Replication factor drops to 1
✗ No fault tolerance
```

---

## 6. CRITICAL SETTINGS TO KNOW

### min.insync.replicas (min.isr)
**Default**: 1
**Your setup should use**: 2

```
Why? With replication-factor=2 and min.isr=2:
✓ Write succeeds only if BOTH brokers acknowledge
✓ Better consistency guarantee
✓ Trade-off: Slower writes, higher availability

If Broker 0 dies:
✗ Writes FAIL (only 1 broker left, need 2)
✓ Reads STILL WORK
```

### acks (Producer Setting)
**Values**: 0, 1, all

```
acks=0 → Fire and forget (fastest, least safe)
acks=1 → Leader acknowledges (medium)
acks=all → All ISR acknowledge (safest, slowest)

Your scenario:
- With min.isr=2 and acks=all
- Write waits for BOTH brokers to confirm
- Guaranteed durability
```

---

## 7. KEY INTERVIEW QUESTION ANSWERS

**Q: You have 2 brokers, 3 partitions, RF=2. Can you lose data?**
```
A: YES, if BOTH brokers fail simultaneously.
   With only 2 brokers and RF=2:
   - Each partition exists on max 2 brokers
   - If both die → data lost
   
   Solution: Use RF=3 (requires 3 brokers)
```

**Q: What's the minimum number of brokers for your setup?**
```
A: 2 brokers minimum (that's your current setup)
   - Replication factor cannot exceed broker count
   - RF=2 requires at least 2 brokers
   - Best practice: RF should be ≤ broker count
```

**Q: If a broker restarts, how do replicas sync?**
```
A: Automatic catch-up:
   1. Broker comes back online
   2. Followers request missing messages from leader
   3. Leader streams all messages since last checkpoint
   4. Follower becomes ISR again
   5. Back to full replication
```

