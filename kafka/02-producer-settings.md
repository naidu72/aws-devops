# Kafka Producer API Settings - Interview Guide

## Most Important Producer Settings

### 1. acks (Acknowledgments)

**What**: How many brokers must confirm before write is successful

```
┌──────────────┬────────────────────────────┬─────────────┬──────────┐
│ acks Value   │ Confirmation Needed        │ Speed       │ Safety   │
├──────────────┼────────────────────────────┼─────────────┼──────────┤
│ 0            │ None (fire & forget)       │ Fastest ⚡  │ Lowest ✗ │
│ 1            │ Leader only                │ Medium 🔄   │ Medium ⚠  │
│ all (or -1)  │ All in-sync replicas (ISR) │ Slowest 🐌  │ Highest✓ │
└──────────────┴────────────────────────────┴─────────────┴──────────┘

For Your Setup (2 brokers, RF=2):
```

```
acks=0 (Risky for Production)
┌─────────────────────────────────────┐
│ Producer sends message               │
│ Returns immediately                  │
│ Doesn't wait for broker              │
│ ⚠️ Network issue? Message lost!      │
└─────────────────────────────────────┘
Use: Only for log analytics (losing data OK)

acks=1 (Medium Safety)
┌─────────────────────────────────────┐
│ Producer sends message               │
│ Leader Broker 0 receives             │
│ Returns success                      │
│ Broker 1 replicates in background    │
│ ⚠️ Leader dies before replicate?     │
│    Message lost!                     │
└─────────────────────────────────────┘
Use: Most common in production

acks=all (Highest Safety - RECOMMENDED for financial/critical data)
┌─────────────────────────────────────┐
│ Producer sends message               │
│ Leader Broker 0 receives             │
│ Waits for Broker 1 to receive        │
│ Returns success only when BOTH ok    │
│ ✓ Data safe on 2 brokers             │
│ ✗ Slower latency                     │
└─────────────────────────────────────┘
Use: Payment systems, critical transactions
```

---

### 2. retries & retry.backoff.ms

**What**: If producer fails, how many times retry + wait between retries

```
retries=3, retry.backoff.ms=100 (milliseconds)

Message send fails
↓
Wait 100ms
↓
Retry 1... fails
↓
Wait 100ms
↓
Retry 2... fails
↓
Wait 100ms
↓
Retry 3... fails
↓
Give up, throw exception
```

**Your setup**: 
- Use `retries=Integer.MAX_VALUE` (keep trying forever)
- Set `delivery.timeout.ms=300000` (5 minutes total timeout)

---

### 3. compression.type

**What**: Compress messages before sending to save bandwidth

```
Compression types:
- none       → No compression (1 MB batch = 1 MB sent)
- snappy     → Fast compression (1 MB → ~400 KB)
- lz4        → Better compression (1 MB → ~300 KB)
- gzip       → Best compression (1 MB → ~200 KB), slower
- zstd       → Modern, good balance

Trade-off:
compression=none   → Faster sends, more network
compression=gzip   → Slower sends, less network

For AWS MSK: Use snappy or lz4 (good balance)
```

---

### 4. batch.size & linger.ms

**What**: Collect multiple messages before sending

```
Default: batch.size=16KB, linger.ms=0

Case 1: batch.size=16KB, linger.ms=0
Message 1: 1 KB → Send immediately ❌ (wait, inefficient)
Message 2: 1 KB → Send immediately ❌
Message 3: 1 KB → Send immediately ❌

Case 2: batch.size=16KB, linger.ms=10ms
Message 1: 1 KB → Wait 10ms
Message 2: 1 KB → Wait 10ms
Message 3: 1 KB → Wait 10ms
[After 10ms] Batch: 3 KB → Send once ✓ (efficient)

For High Throughput:
- batch.size=32768 (32 KB)
- linger.ms=10 (wait 10ms max)
```

---

### 5. buffer.memory

**What**: Total memory for messages waiting to be sent

```
Default: 33554432 bytes (32 MB)

If your app sends too fast:
├─ Messages accumulate in buffer
├─ Buffer fills up (32 MB)
├─ Producer blocks (waits for broker)
└─ App feels slow/frozen

Solution: Increase buffer.memory=67108864 (64 MB)
But: More memory used by producer process
```

---

### 6. max.in.flight.requests.per.connection

**What**: How many unacknowledged requests before producer waits

```
Default: 5

If you need ordered messages (no duplicates):
→ Set to 1 (waits for ack before next send)
→ Guarantees order
→ Trade-off: Slower throughput

If order doesn't matter:
→ Keep at 5-10 (better throughput)
```

---

### 7. enable.idempotence

**What**: Prevent duplicate messages from producer retries

```
Without idempotence (false):
Producer sends Message A
Network timeout
Producer retries Message A
Kafka receives Message A twice
↓
DUPLICATE in topic ❌

With idempotence (true):
Producer sends Message A (with ID: 1)
Network timeout
Producer retries Message A (same ID: 1)
Kafka sees duplicate ID → skips
↓
Message A appears once ✓

For production: ALWAYS use enable.idempotence=true
```

---

### 8. partitioner.class

**What**: Decide which partition message goes to

```
Default: org.apache.kafka.clients.producer.internals.DefaultPartitioner

Strategy 1: Key = null (no key provided)
→ Round-robin across partitions
→ Load balanced
→ No ordering guarantee

Strategy 2: Key = "user123" (same key always)
→ Always goes to same partition (hash based)
→ All messages for user123 ordered
→ Useful for user-specific ordering

Your setup (3 partitions):
Message with key="user123" → always Partition 2
Message with key="user456" → always Partition 0
Message with key=null      → round-robin
```

---

## RECOMMENDED PRODUCER CONFIG FOR YOUR INTERVIEW

```properties
# Connection
bootstrap.servers=b-2.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094,b-1.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094

# Reliability (most important)
acks=all
retries=2147483647  # Max retries
delivery.timeout.ms=300000  # 5 minutes total

# Performance
batch.size=32768  # 32 KB
linger.ms=10  # Wait max 10ms for batch
buffer.memory=67108864  # 64 MB

# Compression
compression.type=snappy

# Ordering & Deduplication
enable.idempotence=true
max.in.flight.requests.per.connection=5

# Partitioning
partitioner.class=org.apache.kafka.clients.producer.internals.DefaultPartitioner
```

---

## Interview Questions about Producer Settings

**Q: What happens if producer acks=all but one broker is down?**
```
A: Write fails. Why?
- You need acks from ALL in-sync replicas (ISR)
- If Broker 0 down, ISR = [Broker 1] only
- Producer waits for Broker 1 to ack
- But acks=all means wait for ALL originally configured
- Timeout after delivery.timeout.ms
- Message send fails
```

**Q: Should I use acks=0?**
```
A: NO (in production with important data).
   Only use for:
   - Log analytics (losing few messages OK)
   - Metrics collection (approximate data OK)
   - Non-critical events
   
   Why risky:
   - No guarantee message reached broker
   - Network blip = data lost
   - Producer returns success falsely
```

**Q: What does enable.idempotence=true do exactly?**
```
A: Prevents duplicate messages from producer retries.
   Without it:
   - Producer timeout → retry
   - Same message sent again
   - Kafka receives twice
   
   With it:
   - Producer assigns sequence number to each message
   - Retried message has same sequence number
   - Kafka broker deduplicates on server side
   - Exactly-once semantics for producer
```

