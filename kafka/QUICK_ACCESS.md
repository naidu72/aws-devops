# Quick Access Guide - Fish Shell

## 🚀 Access Your Study Materials

Your interview prep is ready at:
```
/home/frontier/interview_coverage/kafka/
```

---

## 📖 Read Your Files (Fish Shell Commands)

### Start Here (Your ISR/Replicas Question)
```fish
cat /home/frontier/interview_coverage/kafka/START_HERE_ISR_REPLICAS.md
```

### One Page Cheat Sheet
```fish
cat /home/frontier/interview_coverage/kafka/09-isr-vs-replicas-cheatsheet.md
```

### Visual Diagrams Only
```fish
cat /home/frontier/interview_coverage/kafka/08-isr-replicas-visual-only.md
```

### Detailed Explanations
```fish
cat /home/frontier/interview_coverage/kafka/07-replicas-isr-simple.md
```

---

## 📚 All Files (Complete Course)

```fish
# Table of contents
cat /home/frontier/interview_coverage/kafka/INDEX.md

# Study plan & overview
cat /home/frontier/interview_coverage/kafka/README.md

# Foundation concepts
cat /home/frontier/interview_coverage/kafka/01-msk-basics.md

# Producer API settings
cat /home/frontier/interview_coverage/kafka/02-producer-settings.md

# Consumer API settings
cat /home/frontier/interview_coverage/kafka/03-consumer-settings.md

# Quick reference cheat sheet
cat /home/frontier/interview_coverage/kafka/04-quick-reference.md

# 10 visual diagrams
cat /home/frontier/interview_coverage/kafka/05-visual-diagrams.md

# Your bootstrap & connection
cat /home/frontier/interview_coverage/kafka/06-bootstrap-config.md
```

---

## 🎯 Quick Fish Functions (Add to config)

Add these to `~/.config/fish/config.fish` for quick access:

```fish
# Alias to jump to kafka prep
alias kafkaprep="cd /home/frontier/interview_coverage/kafka && ls -1 *.md"

# Quick view of ISR/Replicas
alias kafkaisr="less /home/frontier/interview_coverage/kafka/START_HERE_ISR_REPLICAS.md"

# View all available files
function kafkafiles
    echo "📚 Kafka Interview Prep Files:"
    ls -1 /home/frontier/interview_coverage/kafka/*.md | sed 's|.*/||' | nl
end

# Search in kafka prep
function kafkasearch --description "Search kafka prep files"
    if test -z "$argv"
        echo "Usage: kafkasearch <search_term>"
        return 1
    end
    grep -r "$argv" /home/frontier/interview_coverage/kafka/
end
```

---

## 📖 Less (Pager) for Better Reading

### Read with better formatting
```fish
less /home/frontier/interview_coverage/kafka/START_HERE_ISR_REPLICAS.md
less /home/frontier/interview_coverage/kafka/09-isr-vs-replicas-cheatsheet.md
```

### Use `j`/`k` to scroll, `q` to quit, `/` to search

---

## 🔍 Search Examples

```fish
# Find all mentions of ISR
grep -n "ISR" /home/frontier/interview_coverage/kafka/*.md

# Find all producer settings
grep -n "acks" /home/frontier/interview_coverage/kafka/02-producer-settings.md

# Find failure scenarios
grep -n "dies" /home/frontier/interview_coverage/kafka/01-msk-basics.md

# List all files with line count
wc -l /home/frontier/interview_coverage/kafka/*.md
```

---

## 📋 Study Checklist (Copy This to Terminal)

```fish
echo "=== ISR & Replicas (15 min) ==="
echo "[ ] Read: START_HERE_ISR_REPLICAS.md"
echo "[ ] Read: 09-isr-vs-replicas-cheatsheet.md"
echo "[ ] Study: 08-isr-replicas-visual-only.md"
echo ""
echo "=== Full Interview Prep (2 hours) ==="
echo "[ ] Read: README.md"
echo "[ ] Read: 01-msk-basics.md"
echo "[ ] Read: START_HERE_ISR_REPLICAS.md"
echo "[ ] Read: 02-producer-settings.md"
echo "[ ] Read: 03-consumer-settings.md"
echo "[ ] Study: 05-visual-diagrams.md"
echo "[ ] Review: 04-quick-reference.md"
echo "[ ] Read: 06-bootstrap-config.md"
echo ""
echo "Ready to crush the interview! 💪"
```

---

## 🎯 Before Interview (30 min prep)

```fish
# Refresh your memory with quick reference
cat /home/frontier/interview_coverage/kafka/04-quick-reference.md

# Review ISR/Replicas one more time
cat /home/frontier/interview_coverage/kafka/09-isr-vs-replicas-cheatsheet.md

# Practice drawing diagrams from
cat /home/frontier/interview_coverage/kafka/05-visual-diagrams.md
```

---

## 💾 Copy to Clipboard (for notes/references)

```fish
# Copy ISR cheat sheet to clipboard
cat /home/frontier/interview_coverage/kafka/09-isr-vs-replicas-cheatsheet.md | pbcopy  # macOS
cat /home/frontier/interview_coverage/kafka/09-isr-vs-replicas-cheatsheet.md | xclip -selection clipboard  # Linux
```

---

## 📱 Quick Questions Answered

**Q: Where are my files?**
```
/home/frontier/interview_coverage/kafka/
```

**Q: I'm confused about ISR - what should I read?**
```
1. START_HERE_ISR_REPLICAS.md (5 min)
2. 09-isr-vs-replicas-cheatsheet.md (5 min)
3. 08-isr-replicas-visual-only.md (5 min)
```

**Q: I have 30 minutes before interview?**
```
1. 04-quick-reference.md (skim it)
2. START_HERE_ISR_REPLICAS.md (read it)
3. 09-isr-vs-replicas-cheatsheet.md (memorize it)
```

**Q: Where are diagrams?**
```
05-visual-diagrams.md (10 diagrams with explanations)
08-isr-replicas-visual-only.md (ISR/Replicas visuals only)
```

**Q: How do I search for something?**
```fish
grep -r "search_term" /home/frontier/interview_coverage/kafka/
# Example:
grep -r "acks" /home/frontier/interview_coverage/kafka/
```

---

## 🎓 Pro Tip: Combined Reading

```fish
# Read multiple files at once with clear separators
echo "=== ISR vs Replicas ===" && cat /home/frontier/interview_coverage/kafka/START_HERE_ISR_REPLICAS.md && echo -e "\n=== Cheat Sheet ===" && cat /home/frontier/interview_coverage/kafka/09-isr-vs-replicas-cheatsheet.md
```

---

## ✅ You're All Set!

**Your prep materials are ready.** 

All 12 comprehensive guides covering:
- ✅ ISR & Replicas (your question)
- ✅ Your 2-broker setup
- ✅ Producer settings
- ✅ Consumer settings
- ✅ Failure scenarios
- ✅ Visuals & diagrams
- ✅ Interview answers
- ✅ Quick reference
- ✅ Complete index

**Next step:** Read `START_HERE_ISR_REPLICAS.md`

**Good luck! 💪**

