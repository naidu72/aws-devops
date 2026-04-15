# Quick Reference - AWS MSK Interview Cheat Sheet

## Your Setup
- **Brokers**: 2
- **Topic**: msk-pprd
- **Partitions**: 3
- **Replication Factor**: 2

---

## 30 Second Explanations

### Partition
- **What**: Divide topic into 3 separate queues
- **Why**: Parallelism (3 different readers can read simultaneously)
- **Example**: Partition 0, Partition 1, Partition 2

### Replication Factor = 2
- **What**: Copy each partition to 2 brokers
- **Why**: If Broker 0 dies, Broker 1 has the data
- **Trade-off**: Slower writes (2 confirmations needed), safer data

### Leader vs Follower
- **Leader**: Handles all reads/writes
- **Follower**: Copies data in background
- **Failover**: When leader dies, follower becomes leader

---

## Critical Scenarios

| Scenario | What Happens | Data Loss? | Availability |
|----------|--------------|-----------|--------------|
| 1 broker dies (2 total) | Failover to other | NO ✓ | YES ✓ |
| Both brokers die | Complete outage | YES ✗ | NO ✗ |
| Network partition | Temporary unavailability | NO (with correct settings) | Recovers |

---

## Producer Settings Ranking (Most Important First)

1. **acks=all** → Wait for all replicas to confirm (safety)
2. **enable.idempotence=true** → Prevent duplicates
3. **retries=∞** → Keep trying on failure
4. **batch.size=32KB** → Combine messages for efficiency
5. **compression.type=snappy** → Reduce network bandwidth

---

## Consumer Settings Ranking (Most Important First)

1. **group.id** → Which group you belong to
2. **auto.offset.reset=earliest** → Start from beginning if new
3. **enable.auto.commit=false** → Manual commits (safer)
4. **isolation.level=read_committed** → Only read committed data
5. **max.poll.records=100** → Don't overwhelm processor

---

## Common Interview Scenarios

### "Broker 0 goes down. Can I still write?"
```
YES ✓
- Partition 0, 2 fail over to Broker 1
- All 3 partitions now on Broker 1
- Writes continue
- But: Only 1 copy left (RF=1), not safe
```

### "What if both brokers die?"
```
NO ✗
- All data lost
- Topic inaccessible
- Disaster!

Prevention: 3+ brokers with RF=3
```

### "How do I guarantee no duplicate messages?"
```
Producer side:
- enable.idempotence=true
- Broker deduplicates on server

Consumer side:
- Process message
- Then commit offset
- If crash before commit, reprocesses (OK)
```

### "Can 2 consumers read the same partition?"
```
NO ✗
- Within same group: NO (partition exclusive)
- Different groups: YES (independent reads)

Rule: Only 1 consumer per partition per group
```

### "What's the fastest way to get all data?"
```
Consumer settings:
- fetch.min.bytes=100000 (100 KB)
- fetch.max.wait.ms=500ms
- max.poll.records=500
- batch.size=32KB
- compression.type=snappy

Trade-off: Fast throughput = Higher latency per message
```

### "What's the safest way to process data?"
```
Producer: acks=all, enable.idempotence=true
Consumer: enable.auto.commit=false, isolation.level=read_committed

Cost: Slower latency (safety tax)
```

---

## ISR (In-Sync Replicas) - Know This!

```
ISR = Replicas caught up with leader

Normal state (RF=2):
Partition 0: Leader on Broker 0, Follower on Broker 1
ISR = [0, 1] ✓ (both up-to-date)

If Broker 1 slow/lagging:
ISR = [0] ⚠️ (only leader, follower behind)

If Broker 1 dies:
ISR = [0] → Still works
       But no fault tolerance

Risk: If Broker 0 dies now, data lost!
```

---

## The "3-2-1" Rule (Not Your Setup!)

```
Ideal production setup:
- 3 brokers (or more)
- RF = 2 (minimum)
- min.isr = 2 (writes require 2 acks)

Why?
- 3 brokers: Quorum majority
- RF=2: Can lose 1 broker
- min.isr=2: Writes safe on 2+ brokers

Your setup (2 brokers, RF=2):
- Borderline: Can lose 1 broker
- Can't lose 2 (data gone)
- Add 3rd broker if possible
```

---

## Key Formulas

```
Max consumers in group = Number of partitions
Your setup: 3 partitions = max 3 consumers

Max data loss tolerance = 1 / (Replication Factor)
RF=2: Can tolerate 1 broker loss
RF=3: Can tolerate 2 broker losses

Throughput = Partitions × (Consumer processing speed)
Your setup: 3 partitions × 1000 msg/sec = 3000 msg/sec max
```

---

## Commands You Need to Know

### Create Topic
```bash
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --command-config $MSK_CONFIG \
  --create --topic msk-pprd \
  --partitions 3 \
  --replication-factor 2
```

### Describe Topic
```bash
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --describe --topic msk-pprd
```
Shows partition distribution and leaders

### Increase Partitions (only 1 direction!)
```bash
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
  --alter --topic msk-pprd \
  --partitions 6  # Increase from 3 to 6
```
⚠️ Cannot decrease partitions

### Produce Message
```bash
echo "Your message" | kafka-console-producer.sh \
  --bootstrap-server $BOOTSTRAP \
  --topic msk-pprd
```

### Consume Messages
```bash
kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP \
  --topic msk-pprd \
  --from-beginning  # Start from partition beginning
```

### Check Consumer Group
```bash
kafka-consumer-groups.sh \
  --bootstrap-server $BOOTSTRAP \
  --group my-group \
  --describe
```
Shows lag (how far behind leader)

---

## Do's and Don'ts

### DO ✓
- ✓ Use RF ≤ broker count
- ✓ Set min.isr = RF - 1
- ✓ Use acks=all for critical data
- ✓ Enable idempotence on producer
- ✓ Commit offsets after processing
- ✓ Monitor consumer lag
- ✓ Start new consumers from latest

### DON'T ✗
- ✗ Use acks=0 for important data
- ✗ Set RF > broker count (fails)
- ✗ Commit before processing
- ✗ Process without partition assignment
- ✗ Have more consumers than partitions
- ✗ Increase min.isr > RF
- ✗ Ignore consumer lag alerts

---

## AWS MSK Specific Notes

### Security
- MSK cluster secured by default
- `$MSK_CONFIG` file contains auth credentials
- TLS enabled on port 9094 (your setup)
- Plain text on port 9092

### Monitoring
- Check AWS CloudWatch for broker metrics
- Monitor consumer lag via CLI
- Check ISR size (should equal RF in normal state)

### Scaling
- Increase partitions: `--partitions 6` ✓
- Decrease partitions: CANNOT ✗
- Add brokers: Via AWS console
- Increase RF: Can do after cluster setup

---

## Last Minute Tips

1. **Always know ISR**: Indicator of cluster health
2. **Understand acks**: Biggest trade-off question
3. **Know failure modes**: What breaks with 1/2/both brokers
4. **Partition math**: 3 partitions = 3× throughput potential
5. **Consumer groups**: Different groups = independent reads
6. **Offsets**: Committed offset = progress marker
7. **Replication**: Copy count = fault tolerance
8. **Rebalancing**: When group composition changes

---

## One More Thing: The Interview Question Pattern

When asked "What happens if X fails?", answer:

1. **What happens to data?** (Lost? Safe? Where?)
2. **What happens to writes?** (Still work? Timeout? Fail?)
3. **What happens to reads?** (Still work? Start from where?)
4. **How long does it take?** (Seconds? Minutes?)
5. **How do I prevent this?** (More brokers? Higher RF? Settings?)

Your setup: 2 brokers, RF=2
→ Can handle 1 broker failure
→ Cannot handle 2 broker failure
→ Need min.isr=2 for safety
→ Use acks=all for durability

