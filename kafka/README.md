# AWS MSK & Kafka Interview Preparation

**Your Interview Setup:**
- 2 AWS MSK Brokers
- Topic: `msk-pprd`
- 3 Partitions
- Replication Factor: 2
- Port: 9094 (TLS)

---

## 📁 File Structure

This interview prep package contains 6 comprehensive guides:

### 1. **01-msk-basics.md** - Start Here First!
**Duration:** 10-15 minutes

Your foundational understanding:
- Visual diagrams of your 2-broker setup
- How 3 partitions distribute across 2 brokers
- Table of partition assignments
- Key concepts (partition, replication, ISR)
- **Two critical failure scenarios:**
  - What happens when Broker 0 dies
  - What happens when Broker 1 dies
- Interview answers to likely questions

**Before any interview, read this 1-2 times to lock in the visuals.**

---

### 2. **02-producer-settings.md** - How to Send Data

**Duration:** 10-15 minutes

Producer API settings you MUST know:

| Setting | Importance | Impact |
|---------|-----------|--------|
| `acks` | 🔴 Critical | Safety vs Speed |
| `retries` | 🔴 Critical | Reliability |
| `enable.idempotence` | 🔴 Critical | Duplicates |
| `batch.size` | 🟡 Important | Throughput |
| `linger.ms` | 🟡 Important | Latency |
| `compression.type` | 🟡 Important | Bandwidth |

**Each setting explained with:**
- What it does (simple)
- Why it matters (trade-offs)
- Your scenario (2 brokers, RF=2)
- Interview Q&A

**You WILL be asked about acks, retries, and idempotence.**

---

### 3. **03-consumer-settings.md** - How to Read Data

**Duration:** 10-15 minutes

Consumer API settings you MUST know:

| Setting | Importance | Impact |
|---------|-----------|--------|
| `group.id` | 🔴 Critical | Group membership |
| `auto.offset.reset` | 🔴 Critical | First read behavior |
| `enable.auto.commit` | 🔴 Critical | Offset management |
| `max.poll.records` | 🟡 Important | Throughput |
| `session.timeout.ms` | 🟡 Important | Rebalancing |
| `isolation.level` | 🟡 Important | Consistency |

**Each setting explained with:**
- What it does
- When it matters
- Why it matters
- Common mistakes
- Interview Q&A

**You WILL be asked about consumer groups and offsets.**

---

### 4. **04-quick-reference.md** - Cheat Sheet

**Duration:** 2-3 minutes (before interview!)

Fast lookup guide:

- 30-second explanations
- Failure scenarios table
- Critical settings ranking
- Common interview scenarios
- ISR explanation
- 3-2-1 rule (production best practices)
- Do's and Don'ts
- One-minute answers to typical questions

**Read this 5 minutes before interview. Perfect for nervous jitters!**

---

### 5. **05-visual-diagrams.md** - See It, Remember It

**Duration:** 5-10 minutes

10 detailed ASCII diagrams showing:
1. Command breakdown
2. Initial partition assignment
3. Message flow (normal)
4. Broker 0 failure
5. Both brokers down (disaster)
6. Latency vs durability trade-off
7. Consumer group partition assignment
8. Offset management
9. ISR visualization
10. Failure recovery timeline

**Great for visual learners. Draw these on whiteboard during interview!**

---

### 6. **06-bootstrap-config.md** - Your Connection

**Duration:** 5 minutes

Understanding your exact setup:
- Bootstrap server breakdown
- Port explanations
- MSK_CONFIG file content
- Connection troubleshooting
- AWS MSK vs Self-Managed
- Fish shell aliases for commands
- Quick command reference

**Interview answer: "I connect to 2 TLS-encrypted brokers at port 9094..."**

---

## 🎯 How to Use These Files

### **First Time Studying (1-2 hours)**
1. Read `01-msk-basics.md` carefully (understand the setup)
2. Read `05-visual-diagrams.md` (see it visually)
3. Read `02-producer-settings.md` (learn producer)
4. Read `03-consumer-settings.md` (learn consumer)
5. Skim `06-bootstrap-config.md` (know your connection)

### **Before Interview (30 minutes)**
1. Read `04-quick-reference.md` (review key points)
2. Review `05-visual-diagrams.md` (remember visuals)
3. Practice answering 5-10 questions from each guide
4. Calm down and remember: you got this!

### **During Interview**
- Draw diagrams on whiteboard (visuals impress)
- Quote failure scenarios (shows you thought through edge cases)
- Explain trade-offs (acks=all vs acks=1)
- Mention ISR (shows you understand replication)
- Say "min.isr=2" (shows production thinking)

### **When Interviewer Asks "What if...?"**
1. Consult failure scenario in mental map
2. Answer: "Data status?", "Write status?", "Read status?", "Recovery time?"
3. Mention prevention: "I would use RF=3 or min.isr=2"
4. Show you think about edge cases

---

## 🔥 Most Important Concepts

### Ranking by Interview Importance

**🔴 If interviewer asks ONE question, be prepared for:**
1. `acks` setting (producer durability)
2. Consumer groups (partitioning strategy)
3. Replication factor (fault tolerance)
4. ISR (in-sync replicas)
5. Broker failure scenarios

**🟡 Likely to come up:**
6. Offset management
7. Latency vs Throughput trade-offs
8. Batch size and compression
9. Consumer lag
10. Rebalancing

**🟢 Nice to know:**
11. min.isr configuration
12. enable.idempotence
13. session.timeout.ms
14. partition.assignment.strategy

---

## 💡 Pro Tips for Interview

### Don't Memorize, Understand
❌ Don't: "acks=0 is fire and forget"
✅ Do: "acks=0 means producer doesn't wait for broker confirmation, so if broker crashes immediately after, the message is lost. I use this only for non-critical metrics."

### Show You've Thought About Edge Cases
❌ Don't: "The cluster handles 3 partitions"
✅ Do: "With 2 brokers and RF=2, I can survive 1 broker failure. If both fail, I lose all data. For production, I'd recommend 3 brokers with RF=3."

### Use Your Actual Setup as Example
❌ Don't: "Kafka clusters typically have..."
✅ Do: "In my setup with 2 brokers and 3 partitions, each partition is replicated to both brokers. When broker 0 dies, partition 0 and 2 fail over to broker 1 because they have replicas there."

### Draw Diagrams (Even if Ugly!)
❌ Don't: "I don't know how to visualize it"
✅ Do: [Draw on whiteboard]
```
Broker 0: P0[L], P1[F], P2[L]
Broker 1: P0[F], P1[L], P2[F]
```

### Mention AWS-Specific Features
❌ Don't: "Kafka has brokers"
✅ Do: "MSK is AWS's managed Kafka. It handles broker provisioning, patching, and monitoring. I connect via TLS on port 9094 using IAM authentication."

---

## ❓ Expected Interview Questions

**Easy (Warmup):**
1. "Tell me about your Kafka setup"
2. "How many brokers do you have?"
3. "What's a partition?"
4. "Why do you need replication?"

**Medium (Real interview):**
5. "What happens if a broker dies?"
6. "How would you increase throughput?"
7. "What does acks=all mean?"
8. "How do consumer groups work?"
9. "What's a consumer lag?"
10. "When would you change batch.size?"

**Hard (They want detailed thinking):**
11. "Design a Kafka system for payments (can't lose data)"
12. "What's a failure scenario where you DO lose data?"
13. "How would you troubleshoot slow consumers?"
14. "Why isn't RF=1 safe?"
15. "Explain the producer reliability vs latency trade-off"

---

## 🚀 Your Elevator Pitch (30 seconds)

Practice saying this smoothly:

"I work with a 2-broker AWS MSK cluster running the topic `msk-pprd` with 3 partitions and replication factor 2. This means each partition is replicated to both brokers, so I can survive 1 broker failure. Producers use `acks=all` for critical data, ensuring both brokers acknowledge before returning. Consumers in groups read partitions in parallel - with 3 partitions, I can have 3 consumers processing simultaneously. If a broker dies, Kafka automatically fails over to the remaining broker."

---

## 📊 Quick Stats for Your Setup

```
Brokers: 2
Partitions: 3
Replication Factor: 2
Max Throughput: ~3000 msg/sec (depends on message size)
Fault Tolerance: Can lose 1 broker
Data Redundancy: 2 complete copies
Min Brokers Needed: 2
Recommended Brokers: 3+ (for HA)
Min ISR: 2 (recommended)
Connection Port: 9094 (TLS)
Region: us-east-1
Cluster Name: cnmskhansenpprd
```

---

## 📚 Study Schedule

**Beginner (First time learning):**
- Hour 1: Read files 01 + 05
- Hour 2: Read files 02 + 03
- Hour 1: Read files 04 + 06
- Hour 1: Practice answering questions

**Intermediate (Refreshing knowledge):**
- 30 min: Skim file 01
- 20 min: Review file 04
- 20 min: Look at file 05 diagrams
- 10 min: Practice top 5 questions

**Day-before-interview:**
- 20 min: Read file 04 (quick reference)
- 15 min: Review failure scenarios in file 01
- 15 min: Look at diagrams in file 05
- 10 min: Calm down and sleep

---

## ✅ Self-Check Before Interview

- [ ] I can draw partition distribution for 2 brokers, 3 partitions, RF=2
- [ ] I know what happens when 1 broker dies (failover, all data safe)
- [ ] I understand acks=0 vs acks=1 vs acks=all (trade-offs)
- [ ] I know producer retries and idempotence prevent duplicates
- [ ] I understand consumer groups and partition assignment
- [ ] I know what offset commit means
- [ ] I can explain ISR (in-sync replicas)
- [ ] I know bootstrap servers and why TLS on 9094
- [ ] I can discuss failure scenarios and prevention
- [ ] I understand throughput limitations (3 partitions max)

**If any unchecked, re-read the relevant file!**

---

## 🎓 Good Luck!

You've got solid knowledge of AWS MSK and Kafka. The key is not to memorize, but to understand the concepts and trade-offs. Think about "why" decisions are made (durability vs speed, safety vs latency) and you'll crush the interview.

**Remember:**
- Draw diagrams (especially partition layout)
- Mention ISR and min.isr
- Talk about failure scenarios
- Think about production concerns (can't lose data)
- Show you understand your specific setup (2 brokers, 3 partitions, RF=2)

**Final tip:** Most interviewers are impressed when you think about edge cases and production implications, not just the happy path.

**Go get that job! 💪**

---

*Interview Prep Package Created: April 2026*
*Your Setup: AWS MSK, 2 Brokers, 3 Partitions, RF=2*
*Bootstrap: b-2,b-1.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094*

