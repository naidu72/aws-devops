# AWS MSK Bootstrap & Configuration Guide

## Your Bootstrap Configuration

```fish
# Fish shell configuration
set -x BOOTSTRAP "b-2.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094,b-1.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094"

# What does this mean?
BOOTSTRAP = Two broker endpoints

b-1.cnmskhansenpprd...amazonaws.com:9094
├─ b-1 = Broker 1 identifier
├─ cnmskhansenpprd = MSK cluster name
├─ xo2b0z = Cluster unique ID
├─ us-east-1 = AWS region
└─ 9094 = TLS-encrypted port

b-2.cnmskhansenpprd...amazonaws.com:9094
└─ Same for Broker 2
```

---

## Port Explanations

```
Port 9092: Plain text (unencrypted)
└─ No TLS/SSL
└─ No authentication required
└─ Use for: Internal testing only

Port 9094: TLS-encrypted (default for MSK)
└─ SSL/TLS encryption
└─ Client certificate required
└─ Use for: Production (your setup)

Port 9096: SASL_SSL (authentication + encryption)
└─ AWS IAM authentication
└─ SASL over SSL
└─ Most secure option
```

---

## Create Topic (Your Command)

```bash
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --command-config $MSK_CONFIG \
                --create --topic msk-pprd \
                --partitions 3 \
                --replication-factor 2

Breaking down:
  $BOOTSTRAP        = 2 broker endpoints
  $MSK_CONFIG       = Config file with:
                      - SSL certificates
                      - Client credentials
                      - Trust store location
  --create          = Action: create new topic
  --topic msk-pprd  = Topic name
  --partitions 3    = Divide into 3 partitions
  --replication-factor 2 = 2 copies each
```

---

## MSK_CONFIG File Content

```properties
# What $MSK_CONFIG file typically contains:

# For TLS (port 9094)
security.protocol=SSL
ssl.truststore.location=/path/to/kafka.client.truststore.jks
ssl.truststore.password=password

# OR for SASL_SSL (port 9096 - AWS IAM)
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

---

## Interview: Understanding Your Setup

**Q: What does the bootstrap server tell Kafka client?**
```
A: Two things:
1. Where to connect initially
   → b-1.cnmskhansenpprd... (primary connection)
   → b-2.cnmskhansenpprd... (fallback)

2. Get cluster metadata
   Client connects to either broker
   Broker responds: "Here are all brokers in cluster"
   Client caches metadata locally
   Future requests go directly to partition leader

Why 2 brokers in bootstrap? (You only have 2 total)
→ Redundancy: If b-1 down, use b-2
→ Load balance: Might connect to either
→ Metadata: Both have full cluster info
```

**Q: Why port 9094 instead of 9092?**
```
A: Security.
9092 = Plain text (no encryption)
9094 = TLS encrypted

MSK default is secured (TLS)
Data in transit encrypted
Certificates verify server identity
Prevents man-in-the-middle attacks

In production AWS: Always use 9094
```

**Q: Can you connect to just 1 broker?**
```
A: YES, technically.
bootstrap.servers=b-1.cnmskhansenpprd...:9094

But NOT recommended:
- If b-1 down: Client can't connect
- Loses redundancy
- Single point of failure

Always provide multiple brokers in bootstrap
```

---

## Common Interview Scenarios

### "Client connects but gets metadata error"
```
Cause: MSK_CONFIG file path wrong
Solution: 
1. Check $MSK_CONFIG exists
2. Check file has valid certificates
3. Check paths are absolute (not relative)
4. Check file permissions (readable)

Debug:
export DEBUG=true
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --command-config $MSK_CONFIG \
                --list
```

### "Connection timeout to broker"
```
Cause: Network/security group issue
Checks:
1. Can ping broker hostname? 
   nslookup b-1.cnmskhansenpprd...
2. Security group allows port 9094?
3. Client has internet access?
4. MSK cluster running in AWS?

Solution:
- Check AWS security groups
- Ensure VPC network connectivity
- Verify MSK cluster status
```

### "Certificate validation failed"
```
Cause: SSL certificate issue
Solutions:
1. SSL truststore path wrong
2. Certificate expired
3. Wrong protocol (SSL vs SASL_SSL)
4. Hostname mismatch

Debug:
openssl s_client -connect b-1.cnmskhansenpprd...:9094

Check: -verify OK
       Subject matches hostname
```

---

## AWS MSK Configuration Best Practices

```
For Production Cluster (Your setup should follow):

1. Broker Configuration
   ├─ 3 brokers minimum (you have 2)
   ├─ Multi-AZ deployment (different availability zones)
   └─ Auto-scaling enabled

2. Security Configuration
   ├─ TLS enabled on port 9094 ✓
   ├─ SASL_SSL for IAM auth (if needed)
   ├─ Private subnets (not public)
   └─ Security group restricts traffic

3. Monitoring
   ├─ CloudWatch metrics enabled
   ├─ Broker CPU < 70%
   ├─ Consumer lag alerts
   └─ Disk usage monitored

4. Backups
   ├─ Automated snapshots
   ├─ Retention policy: 7 days
   └─ Cross-region backup (critical)

5. Topic Configuration
   ├─ Replication factor = 2+ ✓
   ├─ min.insync.replicas = RF-1
   ├─ Retention: 7 days (adjust per use)
   └─ Compression: snappy
```

---

## Interview: AWS MSK vs Self-Managed Kafka

```
Your Setup: AWS MSK (Managed)

Advantages:
✓ AWS handles broker management
✓ Automatic patching/upgrades
✓ Built-in monitoring
✓ Multi-AZ redundancy
✓ Integrated with AWS services
✓ No ops overhead

Disadvantages:
✗ Less control (some settings locked)
✗ Vendor lock-in (AWS only)
✗ Cost higher than self-managed
✗ Limited customization

When to use MSK:
- Production workloads
- Want AWS integration
- Team prefers managed service
- High availability requirement

When NOT to use:
- Learning/testing
- Cost-sensitive
- Custom Kafka version needed
- On-premises infrastructure
```

---

## Setting up Aliases for Fish Shell

```fish
# Add to ~/.config/fish/config.fish

# Set bootstrap servers
set -x BOOTSTRAP "b-2.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094,b-1.cnmskhansenpprd.xo2b0z.c10.kafka.us-east-1.amazonaws.com:9094"

# Set MSK config (if you have one)
set -x MSK_CONFIG "/path/to/msk-client.properties"

# Aliases for common commands
alias kafka-list-topics "kafka-topics.sh --bootstrap-server \$BOOTSTRAP --command-config \$MSK_CONFIG --list"

alias kafka-describe "kafka-topics.sh --bootstrap-server \$BOOTSTRAP --command-config \$MSK_CONFIG --describe --topic"

alias kafka-produce "kafka-console-producer.sh --bootstrap-server \$BOOTSTRAP --topic"

alias kafka-consume "kafka-console-consumer.sh --bootstrap-server \$BOOTSTRAP --from-beginning --topic"

alias kafka-consumer-lag "kafka-consumer-groups.sh --bootstrap-server \$BOOTSTRAP --group"
```

---

## Quick Command Reference

```bash
# List all topics
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --command-config $MSK_CONFIG \
                --list

# Describe your topic
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --command-config $MSK_CONFIG \
                --describe --topic msk-pprd

# Show partition details
# Output shows:
# Topic: msk-pprd
# Partition: 0  Leader: 0  Replicas: [0,1]  ISR: [0,1]
# Partition: 1  Leader: 1  Replicas: [1,0]  ISR: [1,0]
# Partition: 2  Leader: 0  Replicas: [0,1]  ISR: [0,1]

# Alter topic (increase partitions)
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --command-config $MSK_CONFIG \
                --alter --topic msk-pprd \
                --partitions 6  # Increase from 3 to 6

# Delete topic (CAREFUL!)
kafka-topics.sh --bootstrap-server $BOOTSTRAP \
                --command-config $MSK_CONFIG \
                --delete --topic msk-pprd

# Produce message
echo "test message" | kafka-console-producer.sh \
                      --bootstrap-server $BOOTSTRAP \
                      --topic msk-pprd

# Consume messages
kafka-console-consumer.sh --bootstrap-server $BOOTSTRAP \
                          --topic msk-pprd \
                          --from-beginning

# Check consumer groups
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
                         --list

# Check group details
kafka-consumer-groups.sh --bootstrap-server $BOOTSTRAP \
                         --group my-group \
                         --describe
# Shows: topic, partition, current-offset, lag
```

---

## Final Interview Tip

When asked about your bootstrap configuration:

"I connect to 2 MSK brokers using TLS-encrypted port 9094. 
The brokers are in the same cluster (cnmskhansenpprd), 
so when I connect to either one, I get metadata about all 3 partitions 
and both brokers in the cluster. I use the $MSK_CONFIG file 
for SSL certificate validation. This provides redundancy - 
if broker 1 goes down, client automatically connects to broker 2."
```

