# Kafka Consumer API Settings - Interview Guide

## Most Important Consumer Settings

### 1. group.id

**What**: Consumer group identifier

```
Why it matters?
- Consumers in same group share partitions
- Different groups read independently
- Enables parallel processing + durability

Your setup (3 partitions):
┌──────────────────────────────────────────┐
│ Group A: 2 consumers                     │
├──────────────────────────────────────────┤
│ Consumer A1 → Partition 0, 1             │
│ Consumer A2 → Partition 2                │
│ Each partition read by exactly 1 consumer│
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│ Group B: 1 consumer (independent)        │
├──────────────────────────────────────────┤
│ Consumer B1 → Partition 0, 1, 2 (all)   │
│ Reads all 3 partitions independently    │
└──────────────────────────────────────────┘

Rule: Number of consumers ≤ number of partitions
If 4 consumers, 1 will sit idle
```

---

### 2. auto.offset.reset

**What**: What to do if no previous offset found

```
Scenarios:
- New consumer group (no prior history)
- Offset doesn't exist (rolled off after retention)
- Broker has no offset info

┌──────────────┬──────────────────────────────────────┐
│ Value        │ Behavior                             │
├──────────────┼──────────────────────────────────────┤
│ earliest     │ Start from beginning of partition    │
│ latest       │ Start from latest message (skip old) │
│ none         │ Throw exception, fail                │
└──────────────┴──────────────────────────────────────┘

Timeline: (Message 1 to 100 in partition)
earliest → Read 1,2,3...100 ✓ (full replay)
latest   → Read 100 only ✓ (skip backlog)
none     → Exception ✗ (crash)

Interview Tip:
Use auto.offset.reset=earliest for first-time deployments
Use auto.offset.reset=latest for production (only new messages)
```

---

### 3. enable.auto.commit & auto.commit.interval.ms

**What**: Automatically mark consumed messages as processed

```
enable.auto.commit=true (default)
auto.commit.interval.ms=5000

How it works:
Message 1 → Consumed ✓
Message 2 → Consumed ✓
Message 3 → Consumed ✓
[After 5 seconds] → Commit offset to Kafka
Now Kafka knows: up to Message 3 processed

If consumer crashes before commit:
Consumer restarts
Starts from Message 1 again (or last committed)
May reprocess messages 1-3 (at-least-once semantics)
```

**⚠️ RISK: Consumer processes message, crashes before auto-commit**
```
Timeline:
Consumer reads Message 1-10
Processes Message 1-5 ✓
Processing Message 6 → Exception/Crash
Auto-commit triggers for Messages 1-10 (only 1-5 really processed)
Consumer restarts, starts from Message 11
Messages 6-10 never processed ✗ (skipped!)

Solution: Use enable.auto.commit=false (manual commit)
```

---

### 4. manual commit vs auto commit

**AUTO COMMIT (risky for critical data):**
```java
props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000);

While (true) {
    records = consumer.poll(1000);
    for (record : records) {
        process(record);  // If crash here
    }                     // Offset still commits after 5s
}
```

**MANUAL COMMIT (safe, complex):**
```java
props.put("enable.auto.commit", false);

While (true) {
    records = consumer.poll(1000);
    for (record : records) {
        process(record);
    }
    consumer.commitSync();  // Only commit after all processed
}
```

---

### 5. fetch.min.bytes & fetch.max.wait.ms

**What**: When to return data to consumer (batching strategy)

```
Default: fetch.min.bytes=1, fetch.max.wait.ms=500

┌────────────────────────────────────────┐
│ Broker has 3 messages queued           │
├────────────────────────────────────────┤
│ 100ms: 1 message arrives               │
│ 200ms: 2 messages total                │
│ 300ms: 3 messages total                │
│ 500ms: fetch.max.wait.ms triggers      │
│        → Return 3 messages to consumer │
└────────────────────────────────────────┘

fetch.min.bytes=1000  (1 KB minimum)
fetch.max.wait.ms=500

Broker waits until EITHER:
- 1000 bytes available, OR
- 500ms elapsed

Whichever comes first
```

**Trade-off:**
```
fetch.min.bytes=1, fetch.max.wait.ms=10
→ Many small fetches (low latency, network overhead)

fetch.min.bytes=100000, fetch.max.wait.ms=1000
→ Fewer large fetches (high latency, efficient)

For high throughput: Use larger min.bytes
For low latency: Use smaller wait.ms
```

---

### 6. max.poll.records

**What**: Max messages returned per poll() call

```
Default: 500

consumer.poll(timeout);
// Returns UP TO 500 messages

Loop: {
    records = consumer.poll(1000);  // Max 500
    for (record : records) {
        process(record);
        // Takes 200ms per record
        // 500 * 200ms = 100 seconds
    }
}

Problem: Takes 100 seconds to process 500 records
Consumer timeout (session.timeout.ms = 30s default)
Kicked out of group
Rebalancing happens
Reprocess from checkpoint

Solution: Decrease max.poll.records=100
Or increase session.timeout.ms=300000
```

---

### 7. session.timeout.ms & heartbeat.interval.ms

**What**: How long before broker thinks consumer is dead

```
session.timeout.ms=30000  (30 seconds)
heartbeat.interval.ms=3000  (3 seconds)

Timeline:
Second 0: Consumer sends heartbeat ✓
Second 3: Consumer sends heartbeat ✓
Second 6: Consumer sends heartbeat ✓
...
Second 25: Consumer sends heartbeat ✓
Second 28: Consumer CRASHES (no heartbeat)
Second 30: Broker waits 30s, assumes dead
           Rebalancing triggered
           Other consumers takeover

Rule: heartbeat.interval.ms should be 1/3 of session.timeout.ms
Default is good: 3s / 30s
```

---

### 8. partition.assignment.strategy

**What**: How to assign partitions to consumers in group

```
Strategy 1: RangeAssignor (default)
┌────────────────────────────┐
│ Partitions: [0,1,2]        │
│ Consumers: [C1, C2]        │
├────────────────────────────┤
│ C1 → [0, 1] (first 2)      │
│ C2 → [2]    (remaining)    │
│ Result: Uneven ⚠️          │
└────────────────────────────┘

Strategy 2: RoundRobinAssignor
┌────────────────────────────┐
│ Partitions: [0,1,2]        │
│ Consumers: [C1, C2]        │
├────────────────────────────┤
│ C1 → [0, 2] (alternating) │
│ C2 → [1]                   │
│ Result: Better balance ✓   │
└────────────────────────────┘

Strategy 3: StickyAssignor
- Minimizes reassignment during rebalancing
- Keeps assignments when new consumer joins
```

---

### 9. isolation.level

**What**: When to read messages (uncommitted vs committed)

```
isolation.level=read_uncommitted (default)
Reads messages immediately after produced
├─ Faster: See data as soon as possible
└─ Risk: Producer crashes, message rolled back

isolation.level=read_committed
Reads only committed messages
├─ Safer: Only stable data
└─ Slower: Must wait for producer commit

Your scenario (3 partitions, RF=2):
If producer uses acks=all and completes
→ Message is committed
→ Both isolation levels see it

If producer crashes mid-write
→ Message may not be committed
→ read_uncommitted: Sees garbage
→ read_committed: Doesn't see it
```

---

## RECOMMENDED CONSUMER CONFIG FOR YOUR INTERVIEW

```properties
# Connection
bootstrap.servers=b-2.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094,b-1.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094

# Consumer Group
group.id=msk-pprd-consumer-group

# Offset Management
auto.offset.reset=earliest  # Start from beginning if new
enable.auto.commit=false  # Manual commits (safer)

# Fetching
fetch.min.bytes=1024  # 1 KB minimum
fetch.max.wait.ms=500  # Wait max 500ms
max.poll.records=100  # 100 records per poll

# Session Management
session.timeout.ms=30000  # 30 seconds
heartbeat.interval.ms=3000  # 3 seconds

# Isolation
isolation.level=read_committed  # Only committed messages

# Partition Assignment
partition.assignment.strategy=RoundRobinAssignor
```

---

## CONSUMER BEHAVIOR WITH YOUR SETUP (2 brokers, 3 partitions, RF=2)

### Scenario: Broker 0 Dies

```
BEFORE:
Broker 0: Partition 0, 2 (leader)
Broker 1: Partition 1 (leader)
Consumer Group: Reading all 3 partitions ✓

AFTER:
Broker 0: DOWN ✗
Broker 1: Partition 0, 1, 2 (all leaders now)
Consumer Group: Still reading all 3 ✓
  Why? Partition 0,2 failed over to Broker 1
       Data still there (replica on Broker 1)
       Zero message loss (with RF=2)

Consumer Experience:
- Brief hiccup (~5-10 seconds)
- Rebalancing happens
- Consumer catches up
- Continues reading
```

---

## Interview Questions about Consumer Settings

**Q: What's difference between committed and unconsumed offsets?**
```
A: 
Unconsumed offset: Consumer hasn't processed it yet
- Broker has it
- Consumer hasn't called poll() for it
- Could be in network buffer

Committed offset: Consumer processed and committed it
- Consumer called consumer.commitSync()
- Offset stored in Kafka __consumer_offsets topic
- On restart, consumer starts from committed offset

Timeline:
Partition 0: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
                          ↑
                    Unconsumed (not yet polled)
                               ↑
                               Committed offset (already processed)
```

**Q: What happens if a consumer doesn't commit offsets and crashes?**
```
A: Lost progress.
Consumer processed messages 1-50
Didn't commit
Crashed
Restart:
- No committed offset found
- auto.offset.reset=earliest → Start from 1
- Reprocess messages 1-50 (duplicates!)

To prevent:
- Enable manual commit (commitSync())
- Or reduce auto.commit.interval.ms (more frequent)
```

**Q: With 2 brokers, 3 partitions, can 3 consumers read in parallel?**
```
A: YES, perfectly.
Group: [C1, C2, C3]
Partitions: [P0, P1, P2]
Assignment: C1→P0, C2→P1, C3→P2
Each reads independently in parallel ✓

If add 4th consumer:
Group: [C1, C2, C3, C4]
One consumer sits idle ⚠️
Maximum consumers = number of partitions
```

