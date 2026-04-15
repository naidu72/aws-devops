# ISR vs Replicas - One Page Cheat Sheet

## The Difference (ONE SENTENCE)

**Replicas** = brokers with a copy | **ISR** = brokers with the LATEST copy

---

## Your Table Decoded

```
Partition 0:
  Leader Broker = Broker 0
    └─ Who I talk to for reads/writes

  Replica Brokers = [0, 1]
    └─ Broker 0 has a copy, Broker 1 has a copy

  In-Sync Replicas = [0, 1]
    └─ Both Broker 0 and Broker 1 have the LATEST copy


Partition 1:
  Leader Broker = Broker 1
    └─ Who I talk to for reads/writes

  Replica Brokers = [1, 0]
    └─ Broker 1 has a copy, Broker 0 has a copy

  In-Sync Replicas = [1, 0]
    └─ Both Broker 1 and Broker 0 have the LATEST copy


Partition 2:
  Leader Broker = Broker 0
    └─ Who I talk to for reads/writes

  Replica Brokers = [0, 1]
    └─ Broker 0 has a copy, Broker 1 has a copy

  In-Sync Replicas = [0, 1]
    └─ Both Broker 0 and Broker 1 have the LATEST copy
```

---

## Quick Examples

### Example 1: Normal Day
```
Partition 0 on Broker 0 (leader):
Message 1, 2, 3, 4, 5

Partition 0 on Broker 1 (replica):
Message 1, 2, 3, 4, 5

Replica Brokers = [0, 1] ✓
In-Sync Replicas = [0, 1] ✓

Why ISR is [0,1]?
→ Both brokers have messages 1-5 (same data)
```

### Example 2: Broker 1 Slow
```
Partition 0 on Broker 0 (leader):
Message 1, 2, 3, 4, 5, 6

Partition 0 on Broker 1 (replica):
Message 1, 2, 3, 4, 5    ← Missing 6!

Replica Brokers = [0, 1] ✓ (Broker 1 still has a copy)
In-Sync Replicas = [0]    ⚠️ (Broker 1 is lagging)

Why ISR is [0] only?
→ Broker 1 doesn't have message 6 (out of sync)
→ Automatically removed from ISR
```

### Example 3: Broker 0 Dies
```
Before:
Broker 0: Message 1, 2, 3, 4, 5
Broker 1: Message 1, 2, 3, 4, 5
ISR = [0, 1]

After:
Broker 0: DEAD ✗
Broker 1: Message 1, 2, 3, 4, 5 (now leader)
ISR = [1] (Broker 1 takes over)

What happened?
→ Broker 0 died but had no data loss
→ Broker 1 is in ISR, so it can become leader
→ All messages saved!
```

---

## Three Things to Remember

### 1️⃣ Size Comparison

```
Replica size = Should equal RF (replication factor)
               RF=2, so Replicas = 2 brokers

ISR size = Should equal Replica size (in normal state)
           = 2 brokers (when healthy)
           < 2 brokers (when broker is slow/dead)
```

### 2️⃣ ISR Tells You Health

```
ISR = [0, 1]  → Healthy ✓ (both up-to-date)
ISR = [0]     → Degraded ⚠️ (one broker lagging)
ISR = [1]     → One leader left (other died or lagging)
```

### 3️⃣ ISR Decides Failover

```
If leader dies, new leader comes from ISR

Example:
Leader = Broker 0, ISR = [0, 1]
→ If Broker 0 dies, Broker 1 can be leader ✓

But if:
Leader = Broker 0, ISR = [0] (Broker 1 lagging)
→ If Broker 0 dies, no safe broker to promote ✗
→ Partition down, data lost!
```

---

## Interview Answer Template

**Q: What's the difference between replicas and ISR?**

```
A: Replicas are all brokers that have a copy of the partition.
   ISR (In-Sync Replicas) are the replicas that are caught up
   with the leader's latest data.

   In my setup with Partition 0:
   - Replicas = [0, 1] (copies exist on both Broker 0 and 1)
   - ISR = [0, 1] (both have the latest data)

   When Broker 1 gets slow and falls behind:
   - Replicas = [0, 1] (still has a copy somewhere)
   - ISR = [0] (only Broker 0 is caught up)

   The ISR shrinks when a broker lags.
   The ISR is used to decide which broker can become
   the new leader if the current leader fails.
```

---

## Red Flags: When to Worry

```
ISR size drops below RF
→ A broker is lagging or dead
→ You've lost fault tolerance
→ Data at risk if another broker fails

Example in your setup (RF=2):
ISR = [0, 1] ✓ Safe
ISR = [0]    ⚠️ Alert! Need to fix Broker 1

What to check:
- Is Broker 1 running?
- Is Broker 1 responsive?
- Is there network lag?
```

---

## The Order Matters!

```
Replica Brokers = [0, 1]
                   ↑  ↑
                   │  └─ Backup
                   └──── Primary (becomes leader first)

Replica Brokers = [1, 0]
                   ↑  ↑
                   │  └─ Backup
                   └──── Primary (becomes leader first)

Why? Kafka prefers preferred leaders for load balancing.
```

---

## Relationship Chart

```
         REPLICA BROKERS
              /  \
             /    \
            /      \
           /        \
          /          \
    ISR (subset)    Non-ISR
      (caught up)   (lagging)
      ✓ Safe         ✗ Unsafe
```

Example:
```
Replicas = [0, 1] (all brokers with partition copy)
                ↓
    ┌───────────┴───────────┐
    ↓                       ↓
Broker 0               Broker 1
(up-to-date)          (lagging)
  ✓ In ISR              ✗ Not in ISR
```

---

## Bottom Line

| Term | Meaning | Use Case |
|------|---------|----------|
| **Replica** | Brokers with a copy (old or new) | Shows redundancy | 
| **ISR** | Brokers with latest copy | Decides failover |
| **Leader** | Handles reads/writes | Where data goes |

---

## Fish Shell Cheat: Check Your Setup

```fish
# See your partition distribution
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --describe --topic msk-pprd

# Output shows:
# Topic: msk-pprd
# Partition: 0  Leader: 0  Replicas: [0,1]  ISR: [0,1]
# Partition: 1  Leader: 1  Replicas: [1,0]  ISR: [1,0]
# Partition: 2  Leader: 0  Replicas: [0,1]  ISR: [0,1]

# All ISR = [same as Replicas] ✓ HEALTHY
```

