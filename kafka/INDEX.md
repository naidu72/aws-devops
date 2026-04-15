# AWS MSK Kafka Interview Prep - Complete Index

## 📚 All Files Created

### For Your Specific Question: ISR & Replicas

**START HERE:**
1. **START_HERE_ISR_REPLICAS.md** ⭐⭐⭐
   - 5-minute quick guide to your confusion
   - Explains your exact table
   - Interview answer you can copy
   - Read this FIRST if confused

2. **09-isr-vs-replicas-cheatsheet.md** ⭐⭐
   - One-page reference guide
   - Examples with your setup
   - Red flags to watch for
   - Interview template

3. **08-isr-replicas-visual-only.md** ⭐⭐
   - ASCII diagrams
   - Visual examples
   - No long explanations
   - Great for whiteboard practice

4. **07-replicas-isr-simple.md** ⭐
   - Detailed explanations
   - Scenarios with visuals
   - Why ISR matters
   - Full breakdown of your table

---

### Complete Interview Prep Course

5. **README.md** - Start Here (Overview)
   - Study schedule
   - How to use these files
   - Key concepts ranking
   - Pro tips for interview

6. **01-msk-basics.md** - Foundation
   - Your 2-broker setup
   - Partition distribution
   - Failure scenarios
   - Key interview answers

7. **02-producer-settings.md** - Producer API
   - `acks` (most important)
   - `retries` and `idempotence`
   - Latency vs durability trade-offs
   - Recommended config

8. **03-consumer-settings.md** - Consumer API
   - Consumer groups
   - Offset management
   - `auto.offset.reset`
   - Rebalancing

9. **04-quick-reference.md** - Cheat Sheet
   - 30-second explanations
   - Scenarios table
   - Settings ranking
   - Do's and Don'ts

10. **05-visual-diagrams.md** - 10 Detailed Diagrams
    - Topic creation
    - Partition assignment
    - Message flow
    - Failure scenarios
    - ISR visualization

11. **06-bootstrap-config.md** - Your Connection
    - Bootstrap server breakdown
    - MSK_CONFIG explanation
    - Fish shell aliases
    - Troubleshooting

12. **INDEX.md** (this file)
    - Map of everything
    - How to navigate

---

## 📖 Reading Paths

### Path 1: "I'm confused about ISR & Replicas" (Your Question)
```
Time: 15 minutes

1. START_HERE_ISR_REPLICAS.md (5 min)
2. 09-isr-vs-replicas-cheatsheet.md (5 min)
3. 08-isr-replicas-visual-only.md (5 min)

Result: You understand ISR vs Replicas ✓
```

### Path 2: "I want to ace this interview" (Full Prep)
```
Time: 2 hours

1. README.md (10 min) - Overview
2. 01-msk-basics.md (15 min) - Foundation
3. START_HERE_ISR_REPLICAS.md (10 min) - ISR/Replicas
4. 02-producer-settings.md (20 min) - Producer
5. 03-consumer-settings.md (20 min) - Consumer
6. 05-visual-diagrams.md (15 min) - Visuals
7. 04-quick-reference.md (10 min) - Review
8. 06-bootstrap-config.md (10 min) - Your connection

Result: Interview-ready ✓
```

### Path 3: "I have 30 minutes before interview"
```
Time: 30 minutes

1. 04-quick-reference.md (10 min) - Review key points
2. START_HERE_ISR_REPLICAS.md (5 min) - ISR/Replicas
3. 05-visual-diagrams.md (10 min) - Diagrams to draw
4. 09-isr-vs-replicas-cheatsheet.md (5 min) - Interview answer

Result: Refreshed and ready ✓
```

### Path 4: "I want to understand ISR deeply"
```
Time: 45 minutes

1. START_HERE_ISR_REPLICAS.md (5 min)
2. 07-replicas-isr-simple.md (15 min) - Detailed explanations
3. 08-isr-replicas-visual-only.md (10 min) - Visuals
4. 09-isr-vs-replicas-cheatsheet.md (10 min) - Interview answer
5. 01-msk-basics.md (5 min) - Your setup context

Result: Deep understanding ✓
```

---

## 🎯 Quick Navigation

### By Topic

**ISR & Replicas:**
- START_HERE_ISR_REPLICAS.md
- 09-isr-vs-replicas-cheatsheet.md
- 08-isr-replicas-visual-only.md
- 07-replicas-isr-simple.md

**Your Setup (2 brokers, 3 partitions, RF=2):**
- 01-msk-basics.md
- 05-visual-diagrams.md (diagrams 1-4)
- 06-bootstrap-config.md

**Producer Settings (acks, retries, idempotence):**
- 02-producer-settings.md
- 04-quick-reference.md (ranking section)

**Consumer Settings (groups, offsets, lag):**
- 03-consumer-settings.md
- 04-quick-reference.md (ranking section)

**Failure Scenarios (broker dies):**
- 01-msk-basics.md
- 05-visual-diagrams.md (diagrams 4-6)
- 04-quick-reference.md

**Commands & Connection:**
- 06-bootstrap-config.md
- 04-quick-reference.md (command section)

---

## 📊 File Sizes

| File | Lines | Focus |
|------|-------|-------|
| START_HERE_ISR_REPLICAS.md | ~120 | ISR & Replicas quick |
| 09-isr-vs-replicas-cheatsheet.md | ~180 | ISR & Replicas reference |
| 08-isr-replicas-visual-only.md | ~245 | ISR & Replicas visuals |
| 07-replicas-isr-simple.md | ~375 | ISR & Replicas detailed |
| README.md | ~335 | Overview & study plan |
| 01-msk-basics.md | ~195 | Foundation concepts |
| 02-producer-settings.md | ~290 | Producer API |
| 03-consumer-settings.md | ~400 | Consumer API |
| 04-quick-reference.md | ~305 | Quick reference |
| 05-visual-diagrams.md | ~481 | 10 detailed diagrams |
| 06-bootstrap-config.md | ~350 | Connection & commands |

**Total: ~3,271 lines of interview prep** 📚

---

## ✅ Self-Check: Are You Ready?

### ISR & Replicas Specific

- [ ] Can explain the difference in one sentence?
- [ ] Understand your table (Partition 0, 1, 2)?
- [ ] Know what happens if Broker 1 goes slow?
- [ ] Know what happens if Broker 0 dies?
- [ ] Can give interview answer from cheatsheet?

### Complete Interview Ready

- [ ] Know acks=0 vs acks=1 vs acks=all?
- [ ] Understand producer retries & idempotence?
- [ ] Know consumer groups & partition assignment?
- [ ] Understand offset commit & lag?
- [ ] Can draw partition distribution diagram?
- [ ] Know ISR and can explain importance?
- [ ] Understand bootstrap servers & TLS?
- [ ] Can discuss failure scenarios?

---

## 🚀 Interview Day Preparation

### 30 Minutes Before

1. Read: **START_HERE_ISR_REPLICAS.md**
2. Review: **04-quick-reference.md**
3. Study: **09-isr-vs-replicas-cheatsheet.md**
4. Glance: **05-visual-diagrams.md**

### During Interview

- Draw diagrams on whiteboard
- Mention ISR when discussing fault tolerance
- Explain trade-offs (acks, batch size, latency)
- Discuss your specific setup (2 brokers, 3 partitions, RF=2)
- Answer questions with "here's what happens if..." scenarios

### Most Important Interview Topics (By Probability)

1. aks setting
2. Consumer groups
3. Replication factor & ISR
4. Broker failure scenarios
5. Offset management

---

## 📝 Your Exact Setup (Reference)

```
Brokers: 2
Topic: msk-pprd
Partitions: 3
Replication Factor: 2
Port: 9094 (TLS)
Region: us-east-1
Cluster: cnmskhansenpprd
```

**Partition Distribution:**
| Partition | Leader | Replicas | ISR |
|-----------|--------|----------|-----|
| 0 | Broker 0 | [0,1] | [0,1] |
| 1 | Broker 1 | [1,0] | [1,0] |
| 2 | Broker 0 | [0,1] | [0,1] |

---

## 💡 Pro Tips

1. **If confused on ISR**: Read START_HERE_ISR_REPLICAS.md
2. **If need quick answer**: Read 09-isr-vs-replicas-cheatsheet.md
3. **If visual learner**: Study 08-isr-replicas-visual-only.md
4. **If have time**: Read all files in order listed in README.md
5. **Before interview**: Skip to 04-quick-reference.md

---

## 🎓 Good Luck!

You have everything you need. Focus on understanding concepts, not memorizing. Draw diagrams. Think about edge cases. 

**Key Message**: Replicas = copies, ISR = up-to-date copies. Everything flows from that.

Go crush that interview! 💪

