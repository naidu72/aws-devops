# Phase 5: Interview Practice & Mock Scenarios

## Study Duration: Days 5-7
## Topics: Architecture Design, Troubleshooting, Behavioral Questions

---

## Real-World Architecture Design Scenarios

### Scenario 1: E-Commerce Platform

**Requirements:**
- 10,000 concurrent users
- 99.99% uptime (4.38 hours/year downtime)
- Global reach (US, EU, APAC)
- Dynamic inventory & pricing
- Real-time order processing
- Mobile app + web

**Constraints:**
- Budget: $50,000/month
- Team: 8 engineers
- Timeline: 6 months to production

**Design Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│ Global Layer (Route53)                                  │
│ ├─ Failover Policy (Primary: us-east-1, Secondary: eu) │
│ └─ Geolocation Routing (APAC → ap-southeast-1)         │
└────────┬────────────────────────────────────────────────┘
         │
    ┌────┴────────────────────────────────────────┐
    │   Per Region                                │
    │                                             │
    │ ┌──────────────┐  ┌──────────────┐         │
    │ │ CloudFront   │  │ WAF          │         │
    │ │ (CDN)        │  │ DDoS         │         │
    │ └──────┬───────┘  └──────────────┘         │
    │        │                                   │
    │ ┌──────┴────────────────────┐              │
    │ │  API Gateway              │              │
    │ │  (REST + WebSocket)       │              │
    │ └──────┬────────────────────┘              │
    │        │                                   │
    │ ┌──────┴────────────────────────────────┐  │
    │ │  Lambda (Compute)                    │  │
    │ │  ├─ Auth Service                     │  │
    │ │  ├─ Product Service                  │  │
    │ │  ├─ Order Service                    │  │
    │ │  └─ Payment Service                  │  │
    │ └──┬───────────────────────────────┬──┘  │
    │    │                               │     │
    │ ┌──┴─────────┐  ┌─────────┐  ┌────┴──┐ │
    │ │ DynamoDB   │  │RDS (SQL)│  │ ElastiCache
    │ │ (Sessions) │  │(Orders) │  │(Product Cache)
    │ │ (Analytics)│  │(Users)  │  │       │
    │ └────────────┘  └─────────┘  └───────┘ │
    │        │             │           │     │
    │        │             │           │     │
    │ ┌──────┴─────────────┴───────────┴──┐  │
    │ │  S3 (Static Assets, Uploads)      │  │
    │ │  └─ CloudFront distribution       │  │
    │ └───────────────────────────────────┘  │
    │                                         │
    │ ┌───────────────────────────────────┐  │
    │ │ SQS/SNS (Async Processing)        │  │
    │ │ ├─ Order confirmation emails      │  │
    │ │ ├─ Inventory updates              │  │
    │ │ └─ Shipping notifications         │  │
    │ └───────────────────────────────────┘  │
    └─────────────────────────────────────────┘
```

**Design Decisions:**

1. **Compute:**
   - Lambda: Serverless, auto-scales, cost-effective
   - API Gateway: Managed API, authentication
   - Decision: Fits 8-engineer team, no infrastructure ops

2. **Data Layer:**
   - DynamoDB: NoSQL sessions, analytics (horizontal scaling)
   - RDS Multi-AZ: SQL for orders, users (transactions needed)
   - ElastiCache: Redis for product cache (reduce DB load)
   - Decision: Hybrid fits use cases well

3. **High Availability:**
   - Multi-region with Route53 failover
   - Multi-AZ within region (RDS, Lambda, ALB)
   - CloudFront for static content
   - Decision: RPO 5 min, RTO 2 min acceptable for this scale

4. **Cost Optimization:**
   - Lambda pay-per-invocation (no idle costs)
   - DynamoDB on-demand (variable load)
   - Reserved RDS instance (baseline)
   - Decision: ~$30-40k/month estimated

**Monitoring:**
```yaml
# Key Metrics
- API latency (p99 < 500ms)
- Order processing time (< 30s)
- Error rate (< 0.1%)
- DynamoDB throttling (0 throttles)
- RDS CPU (< 80%)
- Lambda cold starts (< 1%)

# Alarms
- Error rate spike > 1%
- Any RDS failover
- CloudFront origin errors
- Payment service timeout
```

**Scaling Considerations:**
- 1M concurrent users: Likely need more Lambda concurrency, RDS read replicas
- Global expansion: Add regions, DynamoDB global tables
- Real-time requirements: Add WebSocket API, Kinesis

**Follow-up Questions (Expected):**
- How would you handle cart abandonment? (async Lambda + SNS)
- How would you prevent overselling inventory? (DynamoDB atomic writes)
- How would you implement recommendations? (ML with SageMaker)
- How would you handle 3x Black Friday traffic? (ASG limits, provisioned capacity)

---

### Scenario 2: Microservices with Service Mesh

**Requirements:**
- 15+ independent microservices
- Each team owns service
- Independent scaling
- Service-to-service authentication
- Observability across services
- Gradual deployments

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│ EKS Cluster (Multi-AZ, 50 nodes)                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────┐          │
│  │ Ingress Controller (ALB)                │          │
│  │ └─ Routes by path/host                  │          │
│  └─────────┬───────────────────────────────┘          │
│            │                                          │
│  ┌─────────┴────────────────────────────┐             │
│  │ Istio Service Mesh                   │             │
│  │ ├─ Envoy sidecars (traffic routing)  │             │
│  │ ├─ mTLS (service-to-service)         │             │
│  │ ├─ Circuit breaking                  │             │
│  │ ├─ Retry logic                       │             │
│  │ └─ Rate limiting                     │             │
│  └─────────┬────────────────────────────┘             │
│            │                                          │
│  ┌─────────┴────────────────────────┐                │
│  │ Microservices (Deployments)      │                │
│  │ ├─ Auth Service (3 pods)         │                │
│  │ ├─ User Service (5 pods)         │                │
│  │ ├─ Order Service (5 pods)        │                │
│  │ ├─ Payment Service (3 pods)      │                │
│  │ ├─ Shipping Service (2 pods)     │                │
│  │ ├─ Notification Service (2 pods) │                │
│  │ └─ ... (more services)           │                │
│  └──────────┬──────────────────────┘                │
│             │                                       │
│  ┌──────────┴────────────────────────────┐          │
│  │ Persistent Storage                   │          │
│  │ ├─ PostgreSQL StatefulSet (3 replicas)         │
│  │ ├─ Redis StatefulSet (cluster mode)  │          │
│  │ └─ S3 via CSI driver                 │          │
│  └──────────────────────────────────────┘          │
│                                                     │
│  ┌────────────────────────────────────────────┐   │
│  │ Observability Stack                       │   │
│  │ ├─ Prometheus (metrics)                   │   │
│  │ ├─ Loki (logs)                            │   │
│  │ ├─ Jaeger (traces)                        │   │
│  │ ├─ Grafana (dashboards)                   │   │
│  │ └─ Alert Manager (alerting)               │   │
│  └────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘

CI/CD Pipeline:
Service Repo → GitHub Actions → Build Docker Image →
  Push to ECR → Update ArgoCD → Flux applies to EKS →
  Canary rollout → Monitor metrics → Automatic promotion
```

**Design Decisions:**

1. **Orchestration: Kubernetes (EKS)**
   - Multi-tenancy: Namespaces per service/team
   - Isolation: Network policies, RBAC
   - Scaling: HPA (horizontal), Cluster autoscaler (nodes)
   - Why: Industry standard for microservices

2. **Service Mesh: Istio**
   - Service-to-service routing (Envoy sidecars)
   - mTLS for security
   - Observability: Automatic metrics, traces
   - Deployment: Canary, blue-green built-in
   - Why: Advanced traffic management without app changes

3. **Observability Stack:**
   - Prometheus: Pull-based metrics
   - Loki: Log aggregation
   - Jaeger: Distributed tracing
   - Grafana: Visualization
   - Why: Complete observability with open-source

4. **CI/CD: GitOps**
   - ArgoCD/Flux: Declarative deployment
   - Git as source of truth
   - Automatic syncing
   - Why: Auditability, rollback via Git

**Multi-Tenancy Strategy:**
```yaml
# Namespace per team
- namespace: team-a
  teams: [alice, bob]
  network-policy: deny all external
  rbac: Team A can manage Team A resources only

- namespace: team-b
  teams: [charlie, david]
  network-policy: deny all external
  rbac: Team B can manage Team B resources only

# Shared: Core services, databases
- namespace: core
  managed-by: platform-team
  network-policy: Allow from team-a, team-b
```

**Scaling Example:**
```
Order spike detected by HPA
  ↓
CPU > 80% for 2 min
  ↓
Scale order service: 5 → 15 pods
  ↓
Metrics show latency still high
  ↓
Cluster autoscaler: 50 → 75 nodes
  ↓
Load normalizes → Scale back down
```

**Follow-up Questions (Expected):**
- How do you handle service versioning? (Istio VirtualServices with routing)
- How do you prevent cascading failures? (Circuit breaker, rate limiting)
- How do you handle cross-team debugging? (Shared observability stack)
- How do you manage database migrations? (Flyway, separate schema per service)

---

### Scenario 3: Data Pipeline Processing

**Requirements:**
- Process 100GB+ daily
- Real-time analytics (<30 sec latency)
- Data retention: 7 years
- Multiple data sources
- Compliance: PII encryption, audit trail

**Architecture:**

```
Data Sources:
├─ API (application events)
├─ Database (change data capture)
├─ Third-party (S3 files)
└─ IoT Devices (MQTT via Kinesis)
  │
  ├─→ Kinesis Data Streams (real-time)
  │   ├─ 100 shards (100MB/sec ingestion)
  │   ├─ Data retention: 24 hours
  │   └─ Consumers: Real-time analytics
  │       ├─ Kinesis Analytics → real-time insights
  │       ├─ Lambda → process → S3 (raw)
  │       └─ Firehose → S3 (batched)
  │
  ├─→ DMS (change data capture)
  │   ├─ Pull from production database
  │   ├─ Transform schema
  │   └─ Write to Redshift (data warehouse)
  │
  └─→ S3 (data lake)
      ├─ Raw layer (original data, compressed)
      │  └─ Lifecycle: Keep 2 years
      ├─ Processed layer (cleaned, deduplicated)
      │  └─ Lifecycle: Keep 7 years
      └─ Analytics layer (aggregated metrics)
          └─ Lifecycle: Keep indefinitely

Processing:
├─ Real-time: Kinesis Analytics
│  └─ Queries on streaming data
│
├─ Batch: Glue jobs (nightly)
│  ├─ Data validation
│  ├─ Deduplication
│  ├─ Privacy masking (PII)
│  └─ Aggregation
│
└─ Ad-hoc: Athena
   └─ SQL queries on S3

Output:
├─ Redshift (analytics dashboard)
├─ OpenSearch (search capability)
├─ QuickSight (business intelligence)
└─ S3 export (compliance/archival)

Monitoring:
├─ Data quality: Row counts, schema validation
├─ Latency: End-to-end processing time
├─ Compliance: PII detection, audit trail
└─ Cost: Data transfer, storage, compute
```

**Design Decisions:**

1. **Ingestion: Kinesis**
   - Real-time streaming
   - Automatic scaling (shards)
   - Multiple consumers
   - Why: Best for streaming data at scale

2. **Storage: S3**
   - Infinite scalability
   - Cost-effective
   - Lifecycle policies for archival
   - Why: Data lake standard

3. **Processing:**
   - Real-time: Kinesis Analytics
   - Batch: Glue (serverless ETL)
   - Ad-hoc: Athena (SQL on S3)
   - Why: Right tool for right job

4. **Analytics:**
   - Redshift: Data warehouse
   - OpenSearch: Full-text search
   - QuickSight: BI dashboard
   - Why: Specialized tools for analytics

**Data Governance:**
```
Schema Registry: Kafka/Confluent
  ├─ Enforce schema evolution
  ├─ Data validation
  └─ Versioning

Data Catalog: Glue Data Catalog
  ├─ Metadata discovery
  ├─ Data lineage
  └─ Ownership

Access Control:
  ├─ IAM for AWS services
  ├─ Redshift database roles
  └─ S3 bucket policies
```

**Cost Optimization:**
```
Daily: 100GB input
  ├─ Raw storage (S3): 500GB × $0.023/GB = $11.50/day
  ├─ Kinesis: 100 shards × $0.36/shard/day = $36/day
  ├─ Glue: 5 jobs × 1 hour × $0.88/DPU/hour = $4.40/day
  └─ Redshift: 2 node → RA3 ($9.26/hour) = $222/day
  
Monthly: ~$7,500
```

**Follow-up Questions (Expected):**
- How do you ensure data consistency? (Idempotent processing, deduplication key)
- How do you handle late-arriving data? (Watermarks, replay mechanism)
- How do you validate data quality? (Deequ/Great Expectations)
- How do you handle schema evolution? (Schema registry versioning)

---

## Troubleshooting Scenarios

### Scenario 1: Application Performance Degradation

**Symptom:** Response time increased from 200ms to 5000ms over 2 hours

**Investigation Steps:**

```
1. Check CloudWatch Dashboard (2 min)
   - Graph latency spike
   - Graph concurrent connections
   - Graph error rate
   - Result: Latency spike started 14:30, no errors

2. Check X-Ray Traces (2 min)
   - Latency by service
   - Database query time
   - External API latency
   - Result: Database queries taking 4000ms (was 50ms)

3. Check RDS Metrics (2 min)
   - CPU utilization: 95% (spike from 30%)
   - Connections: 300 (from 20)
   - Read latency: 4s
   - Result: Database under heavy load

4. Check application logs (2 min)
   - Query volume increased 10x
   - No obvious error pattern
   - Result: Legitimate traffic increase

5. Check Auto Scaling (2 min)
   - ASG size: still 3 instances (should have scaled)
   - CPU alarm threshold: 80%
   - Current CPU: 95% (why didn't it scale?)
   - Result: Scale-up in progress, but slow

6. Check Application Code (5 min)
   - Recent deployment?
   - Yes, deployed 14:25 (5 min before symptom)
   - Query change?
   - Suspect: New query without index?
   - Result: Developer added new report query without INDEX

7. Immediate Action (2 min)
   - Rollback to previous version
   - Latency drops to 200ms
   - Verify user impact: None (rollback was quick)

8. Root Cause Analysis
   - Missing database index on new report query
   - Index should have been applied pre-deployment
   - Process improvement: Add query review step to PR

9. Fix & Prevention
   - Add INDEX to problematic column
   - Add Query Plan check to deployment
   - Add query performance regression test
   - Re-deploy with index
```

**Follow-up Questions:**
- Q: How would you prevent this in future?
  A: Add query plan analysis to code review, database index creation as pre-deployment step, monitoring for query performance regression

- Q: How would you have detected this proactively?
  A: Query execution plan analysis, index usage tracking, slow query log alerts

### Scenario 2: Deployment Failure

**Symptom:** CodePipeline fails at CodeDeploy stage, 5 instances stuck in "Draining" state

**Investigation:**

```
1. Check CodePipeline Status (1 min)
   - Deploy stage: RED
   - CodeDeploy lifecycle event: BeforeAllowTraffic - FAILED
   - Result: Health check failing

2. Check CodeDeploy Logs (2 min)
   - Event: BeforeAllowTraffic
   - Command: /opt/scripts/health-check.sh
   - Exit code: 1 (failure)
   - Output: "Connection refused on port 8080"
   - Result: Application not listening on port 8080

3. SSH into Instance (2 min)
   - Check if process running: ps aux | grep app
   - Result: Process running but listening on port 3000 (not 8080)

4. Check App Configuration (2 min)
   - Environment variable: PORT=3000 (should be 8080)
   - Deployment script set PORT wrong
   - Result: App running but on wrong port

5. Immediate Action (1 min)
   - Stop CodeDeploy deployment (abort)
   - Rollback via ALB: Remove new instances from target group
   - Users back on old version

6. Fix (5 min)
   - Fix deployment script: PORT should be 8080
   - Update CodeDeploy AppSpec health check port
   - Re-deploy

7. Verify (2 min)
   - Health checks passing
   - Instances in "InService"
   - Metrics normal
   - No user impact (quick rollback)

8. Prevention
   - Test health check in staging first
   - Validate port in deployment script
   - Unit test server startup on correct port
```

**Root Cause:** Configuration drift between environment
**Prevention:** Infrastructure as Code, consistent configs across environments

### Scenario 3: Container Crashes in Production

**Symptom:** ECS tasks restarting every 30 seconds, error rate 50%

**Investigation:**

```
1. Check ECS Service Status (1 min)
   - Running count: 0 of 3
   - Stop code: OutOfMemory
   - Result: Tasks killed by OS (OOM)

2. Check Task Definition (1 min)
   - Memory: 512MB
   - Container image: my-app:v2.1.0 (recent change)
   - Result: Recent version might have memory leak

3. Check Task Logs (2 min)
   - CloudWatch log stream
   - Java heap size: 256MB (default)
   - GC logs: Full GC every 5 seconds
   - Result: Memory exhaustion, constant garbage collection

4. Check Previous Version (1 min)
   - v2.0.9: Memory usage 200MB
   - v2.1.0: Memory usage 450MB
   - Result: Memory leak introduced in v2.1.0

5. Immediate Action (1 min)
   - Update task definition: Memory 512MB → 1024MB (temporary)
   - Force new deployment
   - Tasks start successfully
   - Errors drop to 0%

6. Debug Memory Leak (5 min)
   - Compare code changes: v2.0.9 → v2.1.0
   - Suspect: New caching feature not clearing cache
   - Confirm: Memory grows continuously, never released
   - Fix: Add cache eviction policy

7. Verify Fix (5 min)
   - Deploy v2.1.1 with cache fix
   - Memory usage: 220MB (same as v2.0.9)
   - Reduce task memory back to 512MB
   - Confirm stable

8. Prevention
   - Memory profiling in testing
   - Load tests with memory monitoring
   - Unit tests for memory leaks (Java: JProfiler)
   - Application container memory limits in Docker
```

**Root Cause:** Memory leak in new caching feature
**Prevention:** Memory profiling in test environment

---

## Behavioral & Communication Questions

### 1. Complex Deployment Story

**Q: Describe your most complex deployment and lessons learned**

*Sample Answer (3-4 min):*

"At my previous company, we migrated a monolithic Java application to microservices on Kubernetes across two regions. It was complex because:

**Challenges:**
1. 200+ dependencies between services
2. Multiple teams with different languages (Java, Python, Node)
3. Data migration without downtime
4. 99.99% uptime requirement

**What I Did:**
- Led architecture design using domain-driven design
- Implemented strangler pattern: Gradually extracted microservices
- Built centralized logging (ELK stack)
- Implemented service mesh (Istio) for traffic management
- Established observability: Prometheus + Grafana + Jaeger

**Lessons Learned:**
1. Communication is key: Weekly sync-ups between teams prevented rework
2. Test everything: Infrastructure tests caught config errors early
3. Gradual approach: Strangler pattern reduced risk vs big-bang
4. Observability first: Caught issues before they reached users
5. Documentation: Runbooks for team handoff critical

**Outcomes:**
- Reduced deployment time from 4 hours to 15 minutes
- Enabled independent service releases
- Zero downtime migration
- Team autonomy improved significantly"

### 2. On-Call Incident Response

**Q: How do you approach on-call responsibilities and incident response?**

*Sample Answer:*

"I treat on-call as critical responsibility:

**Preparation:**
- Runbooks for common scenarios
- Alert thresholds tuned to avoid false positives
- Team training on incident procedures
- Test incident response quarterly

**Response (First 5 minutes):**
1. Page acknowledges in Pagerduty
2. Assess: Is customer impacted? Severity?
3. Communicate: Slack #incidents channel
4. Triage: What services affected?

**Investigation:**
- Check dashboards and alerts
- Review recent changes
- Check logs for errors
- Use traces to understand flow
- Don't make changes yet (preserve state for analysis)

**Mitigation:**
- Temporary fix if needed (e.g., scale up)
- Or quick rollback if deployment
- Full resolution may take longer

**Post-Incident:**
- Write blameless postmortem (24 hrs)
- Root cause analysis
- Action items to prevent recurrence
- Share lessons with team

**Tools I Use:**
- CloudWatch/Grafana for metrics
- CloudWatch Logs/Datadog for logs
- X-Ray for tracing
- Terraform for safe infrastructure changes"

### 3. Infrastructure Improvement

**Q: Tell about a time you improved infrastructure reliability**

*Sample Answer:*

"At my current role, I noticed database failovers were taking 15 minutes, causing order processing delays.

**Analysis:**
- RDS Multi-AZ was configured but health checks slow
- Monitoring didn't alert on degradation early
- Manual DNS update during failover added delay

**Solution Implemented:**
1. Tuned RDS health checks: Reduced detection time from 30s to 10s
2. Added Route53 health checks with fast failover
3. Implemented automatic DNS failover (no manual intervention)
4. Added detailed monitoring: Connection pool, query latency
5. Created incident runbook and tested procedures

**Results:**
- Failover time reduced: 15 min → 45 seconds
- Zero-downtime failover achieved
- User impact eliminated
- Reduced on-call page response time

**Broader Impact:**
- Team confidence in system reliability increased
- Fewer escalations
- Created template for other databases"

### 4. Staying Updated with AWS

**Q: How do you stay updated with AWS service changes?**

*Sample Answer:*

"I maintain continuous learning through:

**Daily/Weekly:**
- AWS RSS feed for service announcements
- Reddit r/aws for community discussions
- Weekly AWS newsletter

**Monthly:**
- AWS blog deep-dives
- New services review
- Evaluate relevance to current projects

**Quarterly:**
- AWS re:Invent talks (videos released)
- Recertification study (keep certifications current)
- Architecture review: Any new services fit better?

**Applied Learning:**
- Prototype new services in sandbox account
- Build small POCs before production use
- Document learnings for team
- Share in team tech talks

**Recent Examples:**
- Learned about AWS Graviton processors → evaluated cost savings
- Studied EventBridge patterns → refactored to event-driven
- Evaluated new storage classes → optimized S3 costs

I believe in learning by doing, not just reading. The best way to understand a service is to actually build with it in a safe environment first."

---

## Quick-Fire Technical Questions

**Set 1: AWS Fundamentals**

1. What's the difference between Security Groups and NACLs?
   - A: SG stateful/instance-level, NACL stateless/subnet-level

2. Explain VPC peering vs VPN
   - A: Peering = direct network connection, VPN = encrypted tunnel through internet

3. S3 consistency guarantee?
   - A: Strong read-after-write consistency (changed Feb 2020)

4. RDS read replica differences?
   - A: Async replication (lag), can be cross-region, can be promoted

5. When would you use DynamoDB over RDS?
   - A: Unstructured data, massive scale, horizontal partitioning needed

**Set 2: Containerization**

1. What are Docker image layers?
   - A: Each instruction creates read-only layer, final is writable

2. Difference between Deployment and StatefulSet?
   - A: Deployment = stateless (interchangeable), StatefulSet = stateful (stable identity)

3. What's a service mesh?
   - A: Separate layer managing service-to-service communication

4. How does Kubernetes service discovery work?
   - A: DNS lookup (service.namespace.svc.cluster.local) returns ClusterIP, kube-proxy handles forwarding

5. Kubernetes liveness vs readiness probe?
   - A: Liveness = restart if unhealthy, Readiness = remove from service if not ready

**Set 3: CI/CD**

1. What's the purpose of change sets in CloudFormation?
   - A: Preview changes before applying to stack

2. Blue-green vs rolling deployment?
   - A: Blue-green = instant switch (double resources), rolling = gradual (normal resources)

3. What's infrastructure drift?
   - A: Resources manually changed, diverging from IaC

4. How do you manage secrets in CI/CD?
   - A: Use Secrets Manager/Parameter Store, inject at build time

5. What's an artifact repository?
   - A: Central storage for build outputs (Docker images in ECR, JARs in CodeArtifact)

**Set 4: Advanced Topics**

1. Explain RPO and RTO
   - A: RPO = max acceptable data loss, RTO = max acceptable downtime

2. What's observability vs monitoring?
   - A: Monitoring = known metrics, Observability = understand any issue

3. How does X-Ray help debugging?
   - A: Traces request through services, shows latency by segment

4. Reserved Instance vs Spot Instance?
   - A: RI = committed capacity, Spot = up to 90% cheaper but interruptible

5. What's least privilege?
   - A: Grant only minimum permissions needed, explicit deny overrides allow

---

## Final Interview Tips

### Before Interview
- [ ] Review your projects and quantify impact
- [ ] Practice common scenarios (availability, deployment, troubleshooting)
- [ ] Know your tools (AWS CLI, CloudFormation, Terraform)
- [ ] Prepare questions to ask interviewer
- [ ] Research company's tech stack

### During Interview
- **Think out loud:** Explain your reasoning
- **Ask clarifying questions:** "Is cost a concern?" "What's the team size?"
- **Draw diagrams:** Whiteboarding helps communication
- **Start simple:** Then add complexity
- **Mention trade-offs:** No perfect solution
- **Use real examples:** Reference your experience

### Common Mistakes to Avoid
- ❌ Jumping to solution without understanding requirements
- ❌ Claiming expertise in everything
- ❌ Not mentioning testing/monitoring
- ❌ Ignoring security requirements
- ❌ Over-engineering for small requirements
- ❌ Not discussing operations/runbooks
- ❌ Assuming cost is not a factor

### Questions to Ask Interviewer
- "What's your current architecture?"
- "What are the biggest operational challenges?"
- "How do you handle deployments?"
- "What's your incident response process?"
- "How is on-call structured?"
- "What's your DevOps team size?"
- "What are you looking for in this role?"

---

## Mock Interview Script

**Time: 60 minutes**

**0-5 min: Introductions**
"Tell me about yourself and your DevOps journey"

**5-20 min: Architecture Design**
"Design an architecture for [use case - see scenarios above]"
- What services would you use?
- How would you ensure high availability?
- How would you handle scaling?
- What about monitoring/alerting?

**20-35 min: Troubleshooting**
"You have an incident [see troubleshooting scenarios]"
- How would you investigate?
- What tools would you use?
- What's your immediate action?
- How would you prevent this?

**35-50 min: Technical Deep Dive**
"Tell me about a complex project you worked on"
- What challenges did you face?
- How did you solve them?
- What would you do differently?
- What did you learn?

**50-60 min: Questions & Wrap-up**
"What questions do you have for me?"

---

**Total Interview Prep Time: 4-5 hours + ongoing practice**

**You're ready! Good luck with your interview!**
