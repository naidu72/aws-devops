# Phase 4: Advanced Topics & Architecture

## Study Duration: Days 4-5
## Topics Covered: Monitoring/Logging, Security, DR, Cost Optimization

---

## 4.1 Monitoring, Logging & Observability

### Three Pillars of Observability

#### 1. Metrics

**Definition:** Numeric measurement at point in time
- CPU utilization: 65%
- Memory usage: 512 MB
- Requests/sec: 1000
- Latency (p99): 250ms

**CloudWatch Metrics:**
```bash
# Put custom metric
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "ProcessedOrders" \
  --value 100 \
  --unit Count

# Get metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-12345678 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --statistics Average,Maximum,Minimum
```

**Dashboard & Visualization:**
```yaml
# CloudFormation dashboard
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  MyDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: prod-monitoring
      DashboardBody: |
        {
          "widgets": [
            {
              "type": "metric",
              "properties": {
                "metrics": [
                  [ "AWS/EC2", "CPUUtilization", { "stat": "Average" } ],
                  [ "AWS/RDS", "DatabaseConnections" ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "System Metrics"
              }
            }
          ]
        }
```

#### 2. Logging

**Definition:** Events, errors, detailed information

**CloudWatch Logs Setup:**
```bash
# Create log group
aws logs create-log-group --log-group-name /app/production

# Set retention
aws logs put-retention-policy \
  --log-group-name /app/production \
  --retention-in-days 30

# Put log events (application does this)
aws logs put-log-events \
  --log-group-name /app/production \
  --log-stream-name instance-1 \
  --log-events timestamp=1609459200000,message="Application started"
```

**Log Insights Queries:**
```
# Find errors in last hour
fields @timestamp, @message, @logStream
| filter @message like /ERROR/
| stats count() by @logStream

# Performance analysis
fields @duration
| stats avg(@duration), max(@duration), pct(@duration, 99) by @logStream

# Errors by exception type
fields @message
| filter @message like /Exception/
| stats count() by @message
```

**Log Aggregation Architecture:**
```
EC2/ECS/Lambda → CloudWatch Logs
                 ↓
            Log Insights (queries)
                 ↓
            S3 (archive/compliance)
                 ↓
            Athena (query archive)
```

#### 3. Traces (Distributed Tracing)

**Definition:** Request path through distributed system

**X-Ray Integration:**
```python
# Python application with X-Ray
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()  # Patch AWS SDK, HTTP

@xray_recorder.capture('process_order')
def process_order(order_id):
    # This is traced
    db_response = query_database(order_id)
    payment = process_payment(db_response)
    return payment

@xray_recorder.capture('query_database')
def query_database(order_id):
    # Sub-segment shows database query details
    pass
```

**X-Ray Service Map:**
```
    [Client]
      ↓
   [API Gateway]
      ↓
   [Lambda]
      ├→ [DynamoDB]
      ├→ [SNS] → [SQS] → [Worker Lambda]
      └→ [S3]
```

**Debugging with X-Ray:**
- Find slow segments
- Identify failed services
- Database query performance
- External API latency
- Error context

### CloudWatch Alarms

#### Threshold Alarms

```bash
# CPU > 80% for 2 periods
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu-alarm \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alert

# Error rate > 1% for 1 minute
aws cloudwatch put-metric-alarm \
  --alarm-name high-error-rate \
  --metric-name ErrorCount \
  --namespace MyApp \
  --statistic Sum \
  --period 60 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching
```

#### Anomaly Detection

```bash
# Automatically detect anomalies
aws cloudwatch put-metric-alarm \
  --alarm-name response-time-anomaly \
  --comparison-operator LessThanLowerOrGreaterThanUpperThreshold \
  --evaluation-periods 2 \
  --threshold-metric-id e1 \
  --metrics '[
    {
      "id": "m1",
      "returnData": true,
      "metricStat": {
        "metric": {
          "namespace": "MyApp",
          "metricName": "ResponseTime"
        },
        "period": 300,
        "stat": "Average"
      }
    },
    {
      "id": "e1",
      "expression": "ANOMALY_DETECTION_BAND(m1, 2)",
      "returnData": true
    }
  ]'
```

### EventBridge for Event-Driven Monitoring

```bash
# Trigger Lambda on RDS failure
aws events put-rule \
  --name rds-failure-rule \
  --event-pattern '{
    "source": ["aws.rds"],
    "detail-type": ["RDS DB Instance Event"],
    "detail": {
      "EventCategories": ["failure"]
    }
  }'

aws events put-targets \
  --rule rds-failure-rule \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:handle-rds-failure"
```

### Interview Questions Deep Dive

**Q1: Design a comprehensive monitoring strategy for microservices**

*Expected Answer:*
- Metrics: CPU, memory, response time, error rate
- Logs: Centralized logging with aggregation
- Traces: X-Ray for distributed tracing
- Dashboards: Service overview, business metrics
- Alarms: Automated alerting on thresholds
- Anomaly detection: Automatic baseline learning
- SLOs: Define acceptable performance
- Correlation: Link logs/metrics/traces

**Q2: How would you troubleshoot a distributed application issue using logs and traces?**

*Expected Answer:*
1. Alarms triggered → Check CloudWatch dashboard
2. Search logs for errors in affected timeframe
3. Use X-Ray to see service map
4. Find slow/failed segment
5. Correlate with metrics (CPU spike, etc.)
6. Check application logs for context
7. Identify root cause
8. Implement fix
9. Test and deploy

**Q3: Explain the difference between logs, metrics, and traces**

*Expected Answer:*
- Logs: Detailed events (strings, errors)
- Metrics: Numeric measurements (CPU %, latency)
- Traces: Request path through system
- Logs: Best for debugging specific issues
- Metrics: Best for trends and alerting
- Traces: Best for performance analysis
- All three needed for full observability

---

## 4.2 Security Best Practices

### Least Privilege Implementation

```
IAM Policy Pyramid:
        ▲
       / \
      /   \   Specific Resource ARN
     /─────\  & Conditions
    /       \
   /─────────\  Specific Action
  /           \
 /─────────────\ Specific Service
/               \
```

**Example:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSpecificS3Bucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::prod-data",
        "arn:aws:s3:::prod-data/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalAccount": "123456789012"
        },
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T23:59:59Z"
        }
      }
    }
  ]
}
```

### Encryption Strategies

**At Rest:**
```
S3 → SSE-S3 (default) or SSE-KMS (managed keys)
EBS → Encrypted volumes (default enabled)
RDS → Encrypted via KMS
DynamoDB → Encrypted via KMS
```

**In Transit:**
```
TLS 1.2+ for all data transfer
HTTPS for APIs
VPN for on-premises connections
AWS PrivateLink for service access
```

### Secrets Management

**Rotation Strategy:**
```
Old Secret (version 1)
         ↓ (rotation triggered)
New Secret (version 2) created
         ↓ (app still uses v1)
Application gets update → uses v2
         ↓ (after TTL)
Old Secret (version 1) deleted
```

**Implementation:**
```bash
# Create secret with rotation
aws secretsmanager create-secret \
  --name prod/db/password \
  --secret-string '{"username":"admin","password":"InitialPassword"}' \
  --add-replica-regions RegionCode=us-west-2

# Enable automatic rotation
aws secretsmanager rotate-secret \
  --secret-id prod/db/password \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:rotate-secret \
  --rotation-rules AutomaticallyAfterDays=30
```

### CloudTrail Auditing

```bash
# Enable CloudTrail
aws cloudtrail create-trail \
  --name prod-audit \
  --s3-bucket-name prod-audit-logs

aws cloudtrail start-logging --trail-name prod-audit

# Query CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=my-instance \
  --max-results 50

# Analyze in Athena
SELECT eventTime, eventName, userIdentity.principalId, sourceIPAddress
FROM cloudtrail_logs
WHERE eventName = 'DeleteSecurityGroup'
```

### WAF (Web Application Firewall)

```bash
# Create IP set for blocklist
aws wafv2 create-ip-set \
  --name bad-ips \
  --scope CLOUDFRONT \
  --ip-address-version IPV4 \
  --addresses "192.0.2.0/24" "198.51.100.0/24"

# Create web ACL
aws wafv2 create-web-acl \
  --name prod-waf \
  --scope CLOUDFRONT \
  --default-action Allow={} \
  --rules \
    Name=BlockBadIPs,Priority=1,Statement={IPSetReferenceStatement={Arn=arn:...}},Action={Block={}},VisibilityConfig={...}
    Name=RateLimit,Priority=2,Statement={RateBasedStatement={Limit=2000}},Action={Block={}},VisibilityConfig={...}
```

### AWS Security Hub

Centralized view of security issues:
- Compliance checks (CIS, PCI-DSS)
- GuardDuty findings
- CloudTrail events
- Config non-compliance
- Access Analyzer findings

```bash
# Enable Security Hub
aws securityhub enable-security-hub

# Get findings
aws securityhub get-findings \
  --filters "SeverityLabel":{"Value":"CRITICAL","Comparison":"EQUALS"} \
  --max-results 50
```

### Interview Questions Deep Dive

**Q1: Design a security architecture for a multi-tier application**

*Expected Answer:*
- IAM: Least privilege, cross-account if needed
- Network: VPC with public/private subnets, SGS, NACLs
- Encryption: At rest (KMS), in transit (TLS)
- Secrets: Manager/Parameter Store, rotation
- Audit: CloudTrail logging, Config rules
- Monitoring: Security Hub, GuardDuty
- DDoS: WAF, Shield
- Access: MFA, session manager

**Q2: How would you implement secrets rotation?**

*Expected Answer:*
- Use Secrets Manager for automation
- Lambda function handles rotation
- Steps: New version → validate → update app → delete old
- Coordinate with app to handle version changes
- Test rotation in staging first
- Monitor rotation events
- Rollback procedure if rotation fails

**Q3: Explain compliance requirements and how to meet them in AWS**

*Expected Answer:*
- Compliance needs: PCI-DSS, HIPAA, SOC2, etc.
- Config rules: Ensure compliant resources
- CloudTrail: Audit trail for compliance
- Encryption: Data protection requirements
- Access control: IAM for RBAC
- Documentation: Evidence of controls
- Regular audits: Third-party assessments
- Remediation: Process for non-compliant resources

**Q4: Design a vulnerability management and patching strategy**

*Expected Answer:*
- Container scanning: ECR image scan before push
- Host patching: Systems Manager Patch Manager
- Dependency scanning: Application dependencies
- Regular scanning: Automated vulnerability scans
- Prioritization: By severity and exploitability
- Testing: Patches tested before production
- Automation: CodePipeline for patch deployment
- Audit trail: Track all patches applied

---

## 4.3 Disaster Recovery & High Availability

### RPO/RTO Definitions

```
         ┌─────────────────────────────────┐
         │      Last Good State            │
         │      (Your Data)                │
         └──────────────┬──────────────────┘
                        │
                      RPO ◄─── How much data loss acceptable
                        │       (Recovery Point Objective)
                        │
         ┌──────────────┴────────────────┐
         │   Disaster Occurs             │
         │   System Down                 │
         └──────────────┬────────────────┘
                        │
                      RTO ◄─── How long down acceptable
                        │       (Recovery Time Objective)
                        │
         ┌──────────────┴────────────────┐
         │   System Recovered            │
         │   Services Restored           │
         └────────────────────────────────┘

Example: RPO 1 hour = Lose up to 1 hour data
         RTO 4 hours = Service down max 4 hours
```

### Multi-AZ Strategy

```
Current Region (us-east-1)
├── AZ-1a (Primary)
│   ├── EC2 Instance
│   ├── RDS Primary
│   └── ALB
└── AZ-1b (Standby)
    ├── EC2 Instance
    ├── RDS Standby (Multi-AZ)
    └── ALB

Failover: Automatic if AZ-1a fails
```

### Multi-Region Strategy

```
Primary Region (us-east-1)
├── Active-Active replication to
└── Secondary Region (eu-west-1)
    ├── Route53 failover policy
    └── Automatic promotion if primary fails
```

**Implementation:**
```bash
# RDS Multi-Region Read Replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-db-eu-replica \
  --source-db-instance-identifier arn:aws:rds:us-east-1:...:db:prod-db \
  --db-instance-class db.r6i.xlarge \
  --availability-zone eu-west-1a

# Promote to primary (on failover)
aws rds promote-read-replica \
  --db-instance-identifier prod-db-eu-replica
```

### AWS Backup for Centralized Backups

```bash
# Create backup plan
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "prod-plan",
    "Rules": [
      {
        "RuleName": "DailyBackups",
        "TargetBackupVault": "prod-vault",
        "ScheduleExpression": "cron(0 2 ? * * *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 120,
        "Lifecycle": {
          "DeleteAfterDays": 30,
          "MoveToColdStorageAfterDays": 7
        }
      }
    ]
  }'

# Assign resources to backup
aws backup create-backup-selection \
  --backup-plan-id plan-id \
  --backup-selection '{
    "SelectionName": "prod-resources",
    "Type": "RESOURCES",
    "Resources": [
      "arn:aws:ec2:us-east-1:123456789012:volume/*",
      "arn:aws:rds:us-east-1:123456789012:db:prod-db"
    ]
  }'
```

### Disaster Recovery Runbook

```markdown
# DR Runbook: RDS Database Failure

## Detection
- CloudWatch alarm: RDS connection failure
- SNS notification sent
- On-call engineer alerted

## Assessment (5 min max)
- Check RDS status page (AWS)
- Check CloudTrail for recent changes
- Check RDS metrics in CloudWatch
- Try connecting to read replica

## Action Plan

### Option 1: Multi-AZ failover (if configured)
1. Wait for automatic failover (2-3 min)
2. Verify read replica now accepting writes
3. Monitor application connection strings
4. Alert application teams: No action needed

### Option 2: Promote read replica (if different region)
1. Confirm replica caught up: `SELECT COUNT(*) FROM prod_table`
2. Promote: `aws rds promote-read-replica --db-instance-identifier prod-db-replica`
3. Wait 2-5 minutes for promotion
4. Update application connection string
5. Notify application teams

### Option 3: Restore from snapshot
1. `aws rds restore-db-instance-from-db-snapshot --db-instance-identifier prod-db-restored --db-snapshot-identifier snap-xxx`
2. Wait for restoration (10-30 min)
3. Run data validation queries
4. Update application connection string
5. Monitor closely

## Verification
- Query count: `SELECT COUNT(*) FROM transactions WHERE DATE > NOW()-'1 day'::INTERVAL`
- Application metrics: Check request latency, error rates
- Data integrity: Run consistency checks
- Monitor replication lag (if using replicas)

## Post-Recovery
- Root cause analysis: Why did failure happen?
- Improve monitoring/alerting
- Test runbook in staging
- Update documentation
```

### Interview Questions Deep Dive

**Q1: Design a DR strategy with RTO of 4 hours and RPO of 1 hour**

*Expected Answer:*
- Primary Region: us-east-1 (active)
- Secondary Region: eu-west-1 (standby)
- RDS Multi-Region read replica
- Daily automated backups to S3
- Point-in-time recovery enabled (35 day window)
- Route53 failover health check
- On failure: Promote read replica (5 min)
- Total RTO: ~30 min (acceptable for 4 hour RTO)
- RPO: Last backup + transaction logs = 1 hour max

**Q2: How would you test disaster recovery procedures without impacting production?**

*Expected Answer:*
- Use AWS Backup vault for test restore
- Restore to staging/test environment
- Run full system validation
- Simulate failover in staging VPC
- Document recovery time (actual RTO)
- Identify bottlenecks
- Update runbook based on findings
- Regular disaster recovery drills (quarterly)
- Don't test in production environment

**Q3: Explain active-passive vs active-active DR architectures**

*Expected Answer:*
- **Active-Passive:** Primary region active, secondary idle
  - Pros: Simpler, lower cost
  - Cons: Failover lag, less tested
  - RTO: 5-30 min
- **Active-Active:** Both regions serving traffic
  - Pros: Better testing, instant failover
  - Cons: Complex, more expensive, data consistency
  - RTO: Automatic, <1 min
- Recommendation: Start with active-passive, evolve to active-active

**Q4: How do you ensure data consistency in multi-region deployments?**

*Expected Answer:*
- Asynchronous replication (acceptable lag)
- Write to primary only (strong consistency)
- Event-driven replication (DynamoDB streams, RDS logs)
- Read replicas lag a few seconds
- Application logic handles eventual consistency
- Conflict resolution: Last-write-wins or custom logic
- Test data consistency regularly
- Monitor replication lag

---

## 4.4 Cost Optimization

### Reserved Instances vs Savings Plans

**Reserved Instances (RI):**
- Commit to specific instance type/region
- 1-year or 3-year term
- 30-60% savings vs on-demand
- Inflexible if requirements change

**Savings Plans:**
- Commit to hourly spend (not instance)
- Can change instance types/regions
- 20-30% savings
- More flexible than RI

**Compute Savings Plan:**
- Applies to EC2, Fargate, Lambda
- Compute = vCPU hours
- Most flexibility

**Decision Tree:**
```
Predictable workload?
    ├─ YES → Consistent instance type? 
    │         ├─ YES → Reserved Instance
    │         └─ NO → Savings Plan
    └─ NO → Spot Instances / On-demand
```

### Right-Sizing

**Identification:**
```bash
# CPU utilization analysis
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-12345678 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-02-01T00:00:00Z \
  --period 86400 \
  --statistics Average,Maximum

# Memory analysis (requires CloudWatch agent)
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_percent \
  --dimensions Name=InstanceId,Value=i-12345678
```

**Common Issues:**
- t2.large running with <10% CPU → downsize to t3.micro
- m5.xlarge with 100MB RAM used → downsize
- Large instance used only during peaks → use ASG with smaller instances

### Spot Instances for Cost Savings

**Use Cases:**
- Fault-tolerant batch jobs
- Data processing pipelines
- CI/CD builds
- NON-critical workloads

**Savings:** Up to 90% vs on-demand

**Risks:**
- Interruption with 2-min notice
- Limited instance types available
- Price fluctuates

**Implementation:**
```bash
aws ec2 request-spot-instances \
  --spot-price 0.05 \
  --instance-count 10 \
  --type one-time \
  --launch-specification '{
    "ImageId": "ami-12345678",
    "InstanceType": "m5.large",
    "KeyName": "my-key"
  }'
```

### Storage Optimization

**S3 Lifecycle Policy:**
```json
{
  "Rules": [
    {
      "Prefix": "logs/",
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }
  ]
}
```

**Cost Impact:**
```
Day 0: $23/TB (Standard)
Day 30: $12.50/TB (Standard-IA) + retrieval fees
Day 90: $4/TB (Glacier) + retrieval fees
Day 365: $1/TB (Deep Archive) + expensive retrieval
```

### Cost Allocation & Tagging

**Tagging Strategy:**
```json
{
  "Environment": "prod",
  "CostCenter": "engineering",
  "Application": "web-api",
  "Owner": "team-a",
  "Project": "Q1-2024",
  "ManagedBy": "terraform"
}
```

**Cost Analysis:**
```bash
# Group costs by environment
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=ENVIRONMENT
```

### AWS Compute Optimizer

Recommends:
- EC2 right-sizing
- Lambda memory optimization
- RDS instance right-sizing
- EBS volume optimization

```bash
# Get recommendations
aws compute-optimizer get-ec2-instance-recommendations

# Example: t2.large → t2.small (20% cost reduction)
```

### Interview Questions Deep Dive

**Q1: How would you optimize costs for a microservices architecture?**

*Expected Answer:*
- Right-size instances based on actual usage
- Use Savings Plans for predictable workloads
- Spot Instances for batch jobs/non-critical
- Auto-scaling to match demand
- Reserved capacity for databases
- S3 lifecycle policies for archival
- Remove unused resources (EBS snapshots, old RDS)
- Cost allocation tags by service/team
- Regular cost reviews and forecasting

**Q2: Explain Reserved Instances vs Savings Plans**

*Expected Answer:*
- RI: Specific instance type, 30-60% savings
- SP: Flexible instance type, 20-30% savings
- RI: Good if predictable + fixed requirements
- SP: Good if variety of instance types needed
- RI: Can sell unused capacity
- SP: Less liquidatable
- Hybrid: Mix both for optimization

**Q3: How do you balance cost optimization with availability requirements?**

*Expected Answer:*
- Reserved instances for baseline capacity
- Spot instances for burst capacity
- Auto-scaling for flexibility
- Multi-AZ costs vs downtime risk
- Regional costs vs latency
- Right-sizing vs underprovisioning risk
- Monitoring to catch over-sizing
- Cost allocation to teams for accountability

---

## Architecture Patterns

### Serverless Architecture (Lambda-based)

```
API Gateway → Lambda → DynamoDB
             ↓
           S3 (events) → Lambda → SQS → Lambda
           ↓
         CloudFront
```

**Benefits:**
- No servers to manage
- Pay per invocation
- Auto-scales infinitely
- Fast deployment

**Drawbacks:**
- Cold starts
- Limited runtime
- Harder debugging
- Vendor lock-in

### Event-Driven Architecture

```
Event Source → Event Bus (EventBridge) → Multiple Consumers
                                        ├→ Lambda
                                        ├→ SNS
                                        ├→ SQS
                                        └→ Kinesis
```

### Microservices with Service Discovery

```
Load Balancer
    ↓
    ├→ Service A (Port 3000)
    ├→ Service B (Port 3001)
    └→ Service C (Port 3002)

Internal Communication:
Service A → Service Discovery → get Service B IP → call Service B
```

---

**Total Phase 4 Study Time: 5-6 hours**
