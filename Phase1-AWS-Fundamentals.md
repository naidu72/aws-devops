# Phase 1: AWS Fundamentals & Core Services

## Study Duration: Days 1-2
## Topics Covered: IAM, EC2, S3, RDS, Networking

---

## 1.1 Identity & Access Management (IAM)

### Core Concepts

#### 1. IAM Building Blocks

**Users:**
- Individual person with unique credentials (access key ID + secret access key)
- Can have multiple access keys (rotate for security)
- Use cases: Developers, CI/CD service accounts, applications

**Groups:**
- Collection of users
- Policies attached to group apply to all members
- Simplifies permission management at scale
- No group-to-group nesting

**Roles:**
- Set of permissions without permanent credentials
- Temporary security credentials (STS token) with expiration
- Used by: EC2 instances, Lambda, cross-account access, federated users
- Trust policy defines who can assume the role

**Policies:**
- JSON documents defining permissions
- Two types: Identity-based (attached to users/groups/roles) and Resource-based (attached to resources)
- Evaluation: Explicit Deny > Explicit Allow > Default Deny

#### 2. Principle of Least Privilege (PoLP)

Grant only the minimum permissions necessary:
- Start with no permissions
- Add specific permissions needed
- Review and remove unused permissions regularly
- Use resource constraints and conditions

**Example:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::specific-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

#### 3. STS (Security Token Service)

Provides temporary, limited-privilege credentials:
- AssumeRole: Get credentials for a role
- AssumeRoleWithWebIdentity: For web identity federation
- AssumeRoleWithSAML: For SAML federation
- GetSessionToken: MFA-protected temporary credentials

**Credentials include:**
- AccessKeyId
- SecretAccessKey
- SessionToken (proves temporary nature)
- Expiration (default 1 hour, max 1 year)

#### 4. Permission Evaluation Logic

```
1. By default, all requests are denied (Default Deny)
2. Explicit Allow overrides Default Deny
3. Explicit Deny overrides Allow
4. Resource-based and Identity-based policies are additive
5. Permission boundaries set the maximum permissions
6. Session policies further restrict permissions
```

#### 5. Resource-Based vs Identity-Based Policies

**Identity-Based:**
- Attached to IAM principal (user/group/role)
- Specifies what actions are allowed
- Example: S3 read policy on a user

**Resource-Based:**
- Attached to resource (S3 bucket, SQS queue, etc.)
- Specifies who can access and what they can do
- Common in cross-account scenarios
- Example: Trust policy on an IAM role, S3 bucket policy

#### 6. Cross-Account Access Pattern

**Scenario:** Account A (Production) needs to grant Account B (Development) limited EC2 read access

**Step 1: Create role in Account A (Production)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-B-ID:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id"
        }
      }
    }
  ]
}
```

**Step 2: Add permissions to role in Account A**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    }
  ]
}
```

**Step 3: Create policy in Account B (Development)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::ACCOUNT-A-ID:role/CrossAccountRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id"
        }
      }
    }
  ]
}
```

**Step 4: User in Account B assumes role**
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT-A-ID:role/CrossAccountRole \
  --role-session-name development-session \
  --external-id unique-external-id
```

### Interview Questions Deep Dive

**Q1: Explain the difference between IAM Roles and IAM Users**

*Expected Answer Structure:*
- Users: Permanent identities with long-term credentials
- Roles: Temporary identities with temporary credentials via STS
- Users have AccessKeyId/SecretAccessKey (programmatic)
- Roles have AccessKeyId/SecretAccessKey/SessionToken (temporary)
- Roles don't have passwords (only used for API/CLI)
- EC2 instances use roles, not users
- Roles support cross-account access via trust relationships
- Demonstrate understanding of when to use each

**Q2: How would you implement cross-account access for a microservices architecture?**

*Expected Answer Structure:*
- External ID for security (prevents confused deputy)
- Trust policy on role specifies allowed principals
- Resource-based policies for granular control
- Session duration limits
- CloudTrail for auditing access
- CI/CD service account in central account assumes roles in service accounts
- Monitoring and alerting for unusual cross-account access

**Q3: Describe a scenario where you'd use resource-based policies vs identity-based policies**

*Expected Answer Structure:*
- Identity-based: Managing team permissions (attach to IAM user/role)
- Resource-based: S3 bucket policy allowing public read (attach to bucket)
- Resource-based: Cross-account access (principal from different account)
- Resource-based: Service-to-service permissions (Lambda reading SQS)
- Both can work together (intersection of permissions)

**Q4: What is the principle of least privilege and why is it important in DevOps?**

*Expected Answer Structure:*
- Grant minimum necessary permissions
- Limits blast radius if credentials compromised
- Compliance requirement (AWS Well-Architected Framework)
- Reduces risk of accidental resource modifications
- Easier to audit and troubleshoot
- Implementation: Use resource ARNs, conditions, policy boundaries
- Example: Instead of `s3:*`, use `s3:GetObject` on specific bucket

### Practical Exercise 1: EC2 Reading Specific S3 Bucket

**Objective:** Create IAM policy allowing EC2 to read from specific S3 bucket only

**Steps:**

1. **Create S3 bucket and test object**
```bash
aws s3 mb s3://devops-interview-lab-$(date +%s)
aws s3 cp README.md s3://devops-interview-lab-xxxx/
```

2. **Create IAM policy for EC2**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::devops-interview-lab-xxxx/*"
    },
    {
      "Sid": "S3ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::devops-interview-lab-xxxx"
    }
  ]
}
```

3. **Create IAM role**
```bash
aws iam create-role --role-name EC2-S3-Reader \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'

# Attach policy to role
aws iam put-role-policy --role-name EC2-S3-Reader \
  --policy-name S3ReadPolicy \
  --policy-document file://s3-read-policy.json

# Create instance profile and add role
aws iam create-instance-profile --instance-profile-name EC2-S3-Reader
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-Reader \
  --role-name EC2-S3-Reader
```

4. **Launch EC2 instance with role**
```bash
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --iam-instance-profile Name=EC2-S3-Reader
```

5. **SSH into instance and test**
```bash
# Should succeed (can read)
aws s3 cp s3://devops-interview-lab-xxxx/README.md .

# Should fail (no write permissions)
echo "test" > test.txt
aws s3 cp test.txt s3://devops-interview-lab-xxxx/

# Should fail (no access to other buckets)
aws s3 ls s3://other-bucket/
```

**Verification Checklist:**
- ✓ EC2 can read from specific bucket
- ✓ EC2 cannot write to bucket
- ✓ EC2 cannot access other buckets
- ✓ Role includes EC2 service in trust policy
- ✓ Policy uses least privilege (specific actions and resources)

### Practical Exercise 2: Cross-Account Role Assumption

**Objective:** Set up cross-account access where Account B assumes role in Account A

**Account A Setup (Production):**
```bash
# Create role with trust relationship
aws iam create-role --role-name CrossAccountRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::ACCOUNT-B-ID:user/developer"
        },
        "Action": "sts:AssumeRole",
        "Condition": {
          "StringEquals": {
            "sts:ExternalId": "SecureExternalId123"
          }
        }
      }
    ]
  }'

# Add permissions to role
aws iam put-role-policy --role-name CrossAccountRole \
  --policy-name EC2ReadPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "ec2:Describe*",
        "Resource": "*"
      }
    ]
  }'
```

**Account B Setup (Development):**
```bash
# Create policy allowing assumption
aws iam put-user-policy --user-name developer \
  --policy-name AssumeRolePolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "arn:aws:iam::ACCOUNT-A-ID:role/CrossAccountRole",
        "Condition": {
          "StringEquals": {
            "sts:ExternalId": "SecureExternalId123"
          }
        }
      }
    ]
  }'

# User assumes role
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT-A-ID:role/CrossAccountRole \
  --role-session-name dev-session \
  --external-id SecureExternalId123

# Use temporary credentials to access Account A resources
export AWS_ACCESS_KEY_ID=temporary-key
export AWS_SECRET_ACCESS_KEY=temporary-secret
export AWS_SESSION_TOKEN=session-token

aws ec2 describe-instances --region us-east-1
```

**Verification Checklist:**
- ✓ External ID prevents confused deputy problem
- ✓ User in Account B can assume role in Account A
- ✓ Temporary credentials have expiration
- ✓ Access limited to EC2 Describe actions
- ✓ CloudTrail logs cross-account access

### Practical Exercise 3: Inline vs Managed Policies

**Objective:** Create both inline and managed policies, understand differences

**Inline Policy (Attached to specific user):**
```bash
aws iam put-user-policy --user-name developer \
  --policy-name InlineS3Policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::my-bucket/*"
      }
    ]
  }'

# List inline policies
aws iam list-user-policies --user-name developer

# Get policy details
aws iam get-user-policy --user-name developer --policy-name InlineS3Policy
```

**Managed Policy (Reusable):**
```bash
# Create managed policy
aws iam create-policy --policy-name S3ReadManagedPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::my-bucket/*"
      }
    ]
  }'

# Attach to multiple users
aws iam attach-user-policy --user-name developer \
  --policy-arn arn:aws:iam::ACCOUNT-ID:policy/S3ReadManagedPolicy

aws iam attach-user-policy --user-name another-user \
  --policy-arn arn:aws:iam::ACCOUNT-ID:policy/S3ReadManagedPolicy

# Update policy versions
aws iam create-policy-version --policy-arn arn:aws:iam::ACCOUNT-ID:policy/S3ReadManagedPolicy \
  --policy-document file://updated-policy.json \
  --set-as-default
```

**When to Use Each:**

| Aspect | Inline | Managed |
|--------|--------|---------|
| **Reusability** | Single user/role | Multiple users/roles |
| **Versioning** | No version history | 5 versions maintained |
| **Management** | Deleted with principal | Independent lifecycle |
| **Audit Trail** | Per-principal | Centralized |
| **Use Case** | One-off permissions | Standard policies |

---

## 1.2 Compute Services: EC2 & Elastic Load Balancing

### EC2 Fundamentals

#### 1. Instance Types & Selection Criteria

**Naming Convention:** `m5.large`
- `m` = Instance family (general purpose)
- `5` = Generation
- `large` = Size

**Instance Families:**

1. **General Purpose (m, t):**
   - Balanced compute, memory, networking
   - Use cases: Web apps, small-medium DBs, dev/test
   - Examples: m5.large, t3.micro (burstable)

2. **Compute Optimized (c):**
   - High CPU-to-memory ratio
   - Use cases: Batch processing, high-performance web, scientific modeling
   - Examples: c5.large, c5.9xlarge

3. **Memory Optimized (r, x, z):**
   - High memory for data caching/processing
   - Use cases: In-memory databases, caches, SAP, data warehouse
   - Examples: r5.large, x1.32xlarge

4. **Accelerated Computing (p, g, f, inf):**
   - GPU or hardware acceleration
   - Use cases: ML training, graphics rendering, HPC
   - Examples: p3.2xlarge (GPU), g4dn.xlarge (graphics)

5. **Storage Optimized (i, d, h):**
   - High sequential I/O to large data sets
   - Use cases: NoSQL, data warehousing, Elasticsearch
   - Examples: i3.large (SSD), d2.xlarge (HDD)

**Selection Decision Tree:**
```
1. CPU requirements → Compute Optimized vs General Purpose
2. Memory requirements → Memory Optimized vs General Purpose
3. Network requirements → Enhanced networking needed?
4. Cost constraint → t2/t3 burstable for variable workloads
5. Performance requirement → Latest generation (m5 > m4)
```

#### 2. Security Groups & Network ACLs

**Security Groups:**
- Stateful firewall (instance level)
- Default: Deny inbound, Allow outbound
- Return traffic automatically allowed
- Can reference other security groups
- Applied at ENI (Elastic Network Interface)

**Example:** Allow HTTPS from anywhere
```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```
```bash
Internet Request (HTTP) → VPC
    ↓
1. Router (Route Table)
    ↓
2. Network ACL (Subnet Level)
   - Check inbound rules in order
   - Rule 100: ALLOW HTTP 80 from 0.0.0.0/0 ✅
    ↓
3. Security Group (Instance Level)  
   - Check all rules
   - HTTP 80 from 0.0.0.0/0 ✅
    ↓
4. EC2 Instance receives request
    ↓
5. Response sent back
    ↓
6. Security Group (Stateful)
   - Return traffic automatically allowed ✅
    ↓
7. Network ACL (Stateless)
   - Check outbound rules in order
   - Rule 100: ALLOW Ephemeral ports to 0.0.0.0/0 ✅
    ↓
8. Response reaches internet
```
**Network ACLs:**
- Stateless firewall (subnet level)
- Default: Allow all inbound/outbound
- Rules evaluated in order (lowest rule number first)
- Supports deny rules
- Separate inbound and outbound rules

**Difference Table:**

| Feature | Security Group | NACL |
|---------|---|---|
| **Level** | Instance (ENI) | Subnet |
| **Stateful** | Yes | No |
| **Default** | Deny inbound | Allow all |
| **Deny Rules** | No | Yes |
| **Performance** | Faster | Slightly slower |

#### 3. Auto Scaling Groups (ASG)

**Components:**
1. **Launch Template/Configuration:**
   - AMI, instance type, security groups
   - User data script, EBS configuration
   - IAM role, monitoring

2. **Scaling Policies:**
   - **Target Tracking:** Maintain metric at target (e.g., CPU 70%)
   - **Step Scaling:** Scale based on thresholds
   - **Scheduled Scaling:** Scale at specific times

3. **Lifecycle Hooks:**
   - Actions before termination/launch
   - Custom actions before joining load balancer

**Example ASG with Target Tracking:**
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 300
  }'
```

**Health Checks:**
- EC2 status checks (instance reachable)
- ELB health checks (application responding)
- Custom CloudWatch alarms
- Unhealthy instances replaced automatically

#### 4. Elastic Load Balancer Types

**Application Load Balancer (ALB):**
- Layer 7 (Application)
- Use cases: Microservices, container orchestration
- Features: Path-based routing, hostname-based routing, advanced rules
- Supported protocols: HTTP, HTTPS, HTTP/2, WebSocket

**Routing Example:**
```bash
# Path-based routing
/api/* → api-service
/static/* → static-service

# Hostname-based routing
api.example.com → api-service
www.example.com → web-service
```

**Network Load Balancer (NLB):**
- Layer 4 (Transport)
- Use cases: Extreme performance, non-HTTP protocols
- Throughput: >1M requests/sec, ultra-low latency
- Protocols: TCP, UDP, TLS, TCP_UDP

**Use Case Comparison:**
```
ALB: 
- Microservices with many targets
- Content-based routing needed
- HTTP/HTTPS workloads

NLB:
- Extreme performance (gaming, real-time)
- Non-HTTP protocols (DNS, MQTT)
- IoT applications
- Millions of requests per second
```

**Classic Load Balancer (CLB):**
- Older generation, not recommended
- Some legacy applications only option

#### 5. Target Groups & Health Checks

**Target Group Configuration:**
```bash
aws elbv2 create-target-group \
  --name my-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-12345678 \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2
```

**Health Check Logic:**
1. Request sent to health check path every interval
2. If response code matches expected (default 200), marked healthy
3. After `healthy-threshold-count` successes, added to rotation
4. After `unhealthy-threshold-count` failures, removed from rotation
5. Failed instances in ASG replaced

``` bash
To push health check logs to S3
aws elbv2 modify-load-balancer-attributes \                                                                                      --load-balancer-arn <trgetgroup ARN> \
    --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=alb_targetgroup_log Key=access_logs.s3.prefix,Value=ALB-logs

s3 bucket permission to push the log as below 
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<account_number>:root"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::alb_targetgroup_log/ALB-logs/AWSLogs/account_number/*"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "delivery.logs.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::alb_targetgroup_log/ALB-logs/AWSLogs/account_number/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::127311923021:root"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::alb_targetgroup_log"
        }
    ]
}


```
**Common Issues:**
- Application not listening on configured port
- Security group blocking health check traffic
- Health check path returning wrong status code
- Application startup slower than health check timeout

``` bash
Internet
    ↓
┌─────────────────────────────────────────────────────────────┐
│                    Internet Gateway                          │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│                 Application Load Balancer                   │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │  Public Subnet  │    │  Public Subnet  │                │
│  │   AZ-1a         │    │   AZ-1b         │                │
│  └─────────────────┘    └─────────────────┘                │
│                                                             │
│  Target Group: WebServer-TG                                 │
│  ├─ Target: i-1234567890abcdef0 (10.0.3.10:80)            │
│  ├─ Target: i-0987654321fedcba0 (10.0.4.15:80)            │
│  ├─ Target: i-abcdef1234567890 (10.0.3.25:80)             │
│  └─ Target: i-fedcba0987654321 (10.0.4.30:80)             │
└─────────────────────────────────────────────────────────────┘
    ↓ (Direct traffic routing to instances)
┌─────────────────────────────────────────────────────────────┐
│                 Auto Scaling Group                          │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ Private Subnet  │    │ Private Subnet  │                │
│  │   AZ-1a         │    │   AZ-1b         │                │
│  │  ┌───────────┐  │    │  ┌───────────┐  │                │
│  │  │ EC2 Web   │  │    │  │ EC2 Web   │  │                │
│  │  │ Server 1  │  │    │  │ Server 2  │  │                │
│  │  │10.0.3.10  │  │    │  │10.0.4.15  │  │                │
│  │  └───────────┘  │    │  └───────────┘  │                │
│  │  ┌───────────┐  │    │  ┌───────────┐  │                │
│  │  │ EC2 Web   │  │    │  │ EC2 Web   │  │                │
│  │  │ Server 3  │  │    │  │ Server 4  │  │                │
│  │  │10.0.3.25  │  │    │  │10.0.4.30  │  │                │
│  │  └───────────┘  │    │  └───────────┘  │                │
│  └─────────────────┘    └─────────────────┘                │
└─────────────────────────────────────────────────────────────┘


This is for ALB flow 
┌─────────────────────────────────────────────────────────────┐
│              Application Load Balancer                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Target Group                         │   │
│  │  (This is just a configuration, not a separate     │   │
│  │   physical component)                               │   │
│  │                                                     │   │
│  │  Targets:                                           │   │
│  │  • i-1234567890abcdef0 → 10.0.3.10:80 (Healthy)   │   │
│  │  • i-0987654321fedcba0 → 10.0.4.15:80 (Healthy)   │   │
│  │  • i-abcdef1234567890 → 10.0.3.25:80 (Healthy)    │   │
│  │  • i-fedcba0987654321 → 10.0.4.30:80 (Healthy)    │   │
│  │                                                     │   │
│  │  Health Check: GET /index.html every 30s           │   │
│  │  Load Balancing: Round Robin                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  When request comes in:                                     │
│  1. ALB receives request                                    │
│  2. ALB looks at Target Group configuration                 │
│  3. ALB picks a healthy target                              │
│  4. ALB forwards request DIRECTLY to EC2 instance          │
└─────────────────────────────────────────────────────────────┘
                                ↓ (Direct connection)
┌─────────────────────────────────────────────────────────────┐
│                    EC2 Instances                            │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐│
│  │   EC2-1   │  │   EC2-2   │  │   EC2-3   │  │   EC2-4   ││
│  │10.0.3.10  │  │10.0.4.15  │  │10.0.3.25  │  │10.0.4.30  ││
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘│
└─────────────────────────────────────────────────────────────┘
Real Traffic Flow
Step-by-Step Request Processing:
# 1. User makes request
curl http://my-alb-1234567890.us-east-1.elb.amazonaws.com

# 2. ALB receives request
ALB: "I received a request, let me check my Target Groups"

# 3. ALB checks Target Group configuration
ALB: "Target Group 'WebServer-TG' has these healthy targets:
      - i-1234567890abcdef0 at 10.0.3.10:80 ✅
      - i-0987654321fedcba0 at 10.0.4.15:80 ✅  
      - i-abcdef1234567890 at 10.0.3.25:80 ✅
      - i-fedcba0987654321 at 10.0.4.30:80 ✅"

# 4. ALB selects target (round-robin)
ALB: "I'll send this request to 10.0.3.10:80"

# 5. ALB forwards request DIRECTLY to EC2 instance
ALB → EC2 Instance (10.0.3.10:80)
# No intermediate "Target Group layer" - direct connection!

# 6. EC2 instance responds
EC2 (10.0.3.10) → ALB → User

How ASG Integrates ASG Registration Process
# 1. ASG creates new EC2 instance using Launch Template
ASG: "Creating new instance i-newinstance123 in subnet-private1"

# 2. ASG automatically registers instance with Target Group
ASG → Target Group: "Add i-newinstance123 at 10.0.3.45:80 to WebServer-TG"

# 3. ALB starts health checking the new instance
ALB: "Starting health checks for i-newinstance123"
ALB → EC2 (10.0.3.45): "GET /index.html HTTP/1.1"

# 4. Once healthy, ALB includes it in load balancing
ALB: "i-newinstance123 is healthy, adding to rotation"

# 5. Future requests can now go to the new instance
User Request → ALB → EC2 (10.0.3.45) [New instance now receives traffic]

```
#### 6. EBS Volumes

**Types:**

1. **gp3 (General Purpose SSD):**
   - Default choice for most workloads
   - Baseline 3,000 IOPS, 125 MB/s
   - Max 16,000 IOPS, 1,000 MB/s
   - Cost: Lowest for SSD

2. **io1/io2 (Provisioned IOPS):**
   - High-performance databases
   - Up to 64,000 IOPS (io2), 32,000 (io1)
   - Expensive, for when performance is critical

3. **st1 (Throughput Optimized HDD):**
   - Big data, Hadoop, sequential workloads
   - 500 MB/s sustained throughput
   - Lowest cost for throughput

4. **sc1 (Cold HDD):**
   - Archive, infrequent access
   - Lower throughput (250 MB/s)
   - Lowest cost

**EBS Snapshots:**
- Point-in-time backup
- Incremental (only changed blocks)
- Can create AMI or new volume from snapshot
- Can be shared across regions

```bash
# Create snapshot
aws ec2 create-snapshot --volume-id vol-12345678 \
  --description "Database backup"

# Copy to another region
aws ec2 copy-snapshot --source-region us-east-1 \
  --source-snapshot-id snap-12345678 \
  --destination-region us-west-2

# Create volume from snapshot
aws ec2 create-volume --snapshot-id snap-12345678 \
  --availability-zone us-east-1a
```

#### 7. Instance Metadata Service (IMDSv2)

**IMDSv1 (Vulnerable):**
- Accessible at 169.254.169.254
- No authentication required
- Vulnerable to SSRF attacks

**IMDSv2 (Secure):**
- Requires PUT request for token first
- Token in session headers
- Protected against SSRF

**Enforce IMDSv2:**
```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-12345678 \
  --http-tokens required \
  --http-put-response-hop-limit 1
```

**Usage (IMDSv2):**
```bash
# Get token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Get metadata using token
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/EC2-S3-Reader
```

### Interview Questions Deep Dive

**Q1: What's the difference between ALB and NLB? When would you use each?**

*Expected Answer:*
- ALB: Layer 7, path/host-based routing, microservices
- NLB: Layer 4, extreme performance, non-HTTP protocols
- ALB: Suitable for 10,000+ requests/sec
- NLB: Required for 1M+ requests/sec or millisecond latency
- Example: ALB for REST API, NLB for game server

**Q2: How do you troubleshoot an instance that isn't receiving traffic from a load balancer?**

*Expected Answer:*
- Check target group health status (healthy vs unhealthy)
- Check security group on instance allows load balancer
- Check health check path returns correct status code
- SSH into instance, verify application running on correct port
- Check application logs for errors
- Verify VPC routing correct between ALB subnet and instance subnet
- Use VPC Flow Logs to see if traffic reaching instance

**Q3: Explain the difference between scaling up vs scaling out**

*Expected Answer:*
- Scale up (vertical): Increase instance size (t2.micro → m5.large)
- Scale out (horizontal): Add more instances (1 → 3 instances)
- Scale out: Better for distributed systems, fault-tolerant
- Scale up: Simpler, but hits limits on instance size
- ASG enables scale out automatically
- Modern architectures prefer scale out

**Q4: How would you ensure zero-downtime deployments with Auto Scaling Groups?**

*Expected Answer:*
- Use rolling deployment (ASG gradually replaces instances)
- New instances must pass health checks before old removed
- CodeDeploy AppSpec lifecycle hooks for graceful shutdown
- Long connection draining timeout on load balancer
- Run integration tests on new instances before traffic
- Canary deployment: Route small % of traffic to new version first
- Blue-green: Two parallel stacks, switch at once

---

## 1.3 Storage Services: S3, EBS, EFS

### S3 Fundamentals

#### 1. S3 Buckets & Objects

**Bucket Naming:**
- Globally unique
- 3-63 characters, lowercase, numbers, hyphens
- Cannot start/end with hyphen
- No underscores or dots

**Object Structure:**
- Key: `/path/to/file.txt`
- Version ID (if versioning enabled)
- Metadata: Content-Type, custom headers
- ACL: Permissions on object
- Storage class: How data stored/accessed

**Key Features:**
- 99.999999999% (11 nines) durability
- Multi-AZ replication automatic
- Strong read-after-write consistency
- Unlimited storage

#### 2. S3 Access Control

**Bucket Policies:**
- Resource-based policy
- Applies to whole bucket or specific paths
- Can allow/deny specific principals

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-public-bucket/*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-private-bucket",
        "arn:aws:s3:::my-private-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalAccount": "123456789012"
        }
      }
    }
  ]
}
```

**ACLs (Access Control Lists):**
- Object-level permissions
- Predefined grantees: Owner, AuthenticatedUsers, Everyone
- Less flexible than bucket policies
- Legacy (bucket policies preferred)

**Object Ownership:**
- Controls who can change ACLs
- ACL vs Bucket Owner Enforced (recommended)
- Prevents accidental public exposure

#### 3. S3 Storage Classes & Lifecycle Policies

**Storage Classes:**

1. **S3 Standard:**
   - Default, frequently accessed data
   - Instant retrieval
   - Highest cost

2. **S3 Standard-IA (Infrequent Access):**
   - Infrequent access but rapid when needed
   - Cheaper but per-GB retrieval fee
   - Min 30-day storage

3. **S3 One Zone-IA:**
   - Single AZ (data loss risk)
   - Cheaper than Standard-IA
   - Minimum 30 days

4. **S3 Glacier Instant:**
   - Archive data, retrieved within milliseconds
   - 90-day minimum storage

5. **S3 Glacier Flexible:**
   - Archive, retrieval in minutes to hours
   - Expedited (1-5 min), Standard (3-5 hrs), Bulk (5-12 hrs)
   - Very cheap storage

6. **S3 Deep Archive:**
   - Long-term archive (7-10 years)
   - Retrieval 12 hours
   - Cheapest

**Lifecycle Policy Example:**
```json
{
  "Rules": [
    {
      "Id": "Archive old data",
      "Status": "Enabled",
      "Prefix": "documents/",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    }
  ]
}
```

**Timeline Behavior:**
- Day 0: Upload to S3 Standard
- Day 30: Transition to Standard-IA (1GB cost: $0.0125)
- Day 90: Transition to Glacier (1GB cost: $0.004)
- Day 365: Delete

#### 4. S3 Encryption

**SSE-S3 (Server-Side Encryption with S3 Managed Keys):**
- AWS manages keys
- AES-256 encryption
- Default when Encrypt is enabled
- Simplest option

**SSE-KMS (Server-Side Encryption with KMS):**
- Customer master key (CMK) managed by KMS
- Audit trail of key usage
- Fine-grained access control
- Per-request cost for key operations

**Client-Side Encryption:**
- Encrypt before uploading
- AWS never sees unencrypted data
- Most secure
- Application responsible for keys

**Implementation:**
```bash
# SSE-S3
aws s3 cp file.txt s3://bucket/file.txt \
  --sse AES256

# SSE-KMS
aws s3 cp file.txt s3://bucket/file.txt \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678

# Enable default encryption on bucket
aws s3api put-bucket-encryption --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  }'
```

#### 5. S3 Versioning & Cross-Region Replication

**Versioning:**
- Multiple versions of same object
- Cannot be disabled, only suspended
- Adds version ID to each object
- More storage used
- Enables rollback

```bash
# Enable versioning
aws s3api put-bucket-versioning --bucket my-bucket \
  --versioning-configuration Status=Enabled

# List versions
aws s3api list-object-versions --bucket my-bucket

# Delete specific version
aws s3api delete-object --bucket my-bucket \
  --key file.txt --version-id AbCdEfGhIjKlMn
```

**Cross-Region Replication:**
- Automatic copy to another region
- Requires versioning on both buckets
- Replication rules define what's replicated
- Can have different storage class in destination

```bash
# Enable replication
{
  "Role": "arn:aws:iam::123456789012:role/s3-replication",
  "Rules": [
    {
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {
        "Prefix": "documents/"
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::my-bucket-replica",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {
            "Minutes": 15
          }
        },
        "Metrics": {
          "Status": "Enabled"
        }
      }
    }
  ]
}
```

### Interview Questions Deep Dive

**Q1: How would you design a data retention policy for compliance using S3 lifecycle policies?**

*Expected Answer:*
- Identify retention requirements by data type
- Use lifecycle transitions to move old data to cheaper storage
- Use Object Lock or Glacier Deep Archive for immutable backups
- Set expiration date when data no longer needed
- Log lifecycle events in CloudTrail
- Test policy in non-production first
- Monitor storage costs regularly

**Q2: Explain the differences between EBS, EFS, and S3. When would you use each?**

*Expected Answer:*
- **EBS:** Instance storage, block-level, single AZ, persistent
- **EFS:** Shared storage, NFS protocol, multi-AZ, elastic
- **S3:** Object storage, unlimited scale, REST API, eventual consistency
- EBS: Database volumes, OS disks
- EFS: Shared data across instances, NFS mount
- S3: Backups, archives, static content

**Q3: What are the implications of S3 strong read-after-write consistency?**

*Expected Answer:*
- Application can immediately read after writing
- No eventual consistency delays
- Simplifies application logic
- Improved for versioning scenarios
- Metadata operations also consistent
- Enables atomic operations for compliance

**Q4: How do you ensure S3 buckets are not exposed to the public accidentally?**

*Expected Answer:*
- Enable S3 Block Public Access (bucket-level and account-level)
- Review bucket policies regularly (deny principle)
- Use versioning to recover from accidental deletion
- Enable CloudTrail logging for audit trail
- Use automated scanning (AWS Config)
- Implement least privilege access
- Regular security review process

---

## 1.4 Database Services: RDS & DynamoDB

### RDS Fundamentals

#### 1. RDS Deployment Models

**Single-AZ:**
- Single database instance in one AZ
- No automatic failover
- Suitable for: Dev/test, non-critical workloads
- Lower cost
- Risk: If AZ fails, database unavailable

**Multi-AZ:**
- Primary in one AZ, standby in another
- Automatic failover if primary fails
- Synchronous replication
- Slight latency increase during replication
- Production recommended
- ~double the cost

```bash
# Create Multi-AZ RDS instance
aws rds create-db-instance \
  --db-instance-identifier my-database \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password MyPassword123 \
  --allocated-storage 20 \
  --multi-az
```

#### 2. Read Replicas

**Purpose:**
- Scale read capacity
- Geographic distribution
- Zero RPO (Recover Point Objective)
- Can be in different region

**Async Replication:**
- Changes replicated asynchronously
- Slight replication lag possible
- Read replica can lag behind primary
- Application must handle eventual consistency

**Promotion:**
- Read replica can be promoted to standalone DB
- Becomes independent instance
- Breaks replication relationship
- Use case: Disaster recovery failover

```bash
# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-database-read \
  --source-db-instance-identifier my-database

# Cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier my-database-read-west \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:my-database \
  --db-instance-class db.t3.micro

# Promote read replica
aws rds promote-read-replica \
  --db-instance-identifier my-database-read
```

#### 3. Backup & Restore Strategies

**Automated Backups:**
- Daily automated snapshots (configurable backup window)
- Transaction logs retained for point-in-time recovery
- Retention: 1-35 days (default 7)
- Can restore to any point within retention period

**Manual Snapshots:**
- On-demand backups
- Not automatically deleted
- Use before major changes
- Copy to other region for DR

**Restore Options:**
- Restore to new instance (cannot overwrite existing)
- Point-in-time recovery (PITR) within retention period
- Restore from snapshot

**RPO/RTO Calculation:**
- RPO = Backup frequency + transaction log retention
- RTO = Restore time + application startup time
- Example: With hourly backups, RPO = 1 hour max

```bash
# Create manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier my-database \
  --db-snapshot-identifier my-database-backup-20240101

# List available backups
aws rds describe-db-backups \
  --db-instance-identifier my-database

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-database-restored \
  --db-snapshot-identifier my-database-backup-20240101

# Restore to specific point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier my-database \
  --target-db-instance-identifier my-database-pitr \
  --restore-time 2024-01-01T12:00:00Z
```

#### 4. Parameter Groups & Performance Tuning

**Parameter Groups:**
- Control database behavior (memory, max connections, etc.)
- Changes can require instance restart
- Dynamic parameters: Apply immediately
- Static parameters: Require restart

**Common Parameters:**
- `max_connections`: Max database connections
- `shared_buffers`: Memory for caching
- `log_queries_not_using_indexes`: Performance troubleshooting
- `slow_query_log`: Log queries > threshold

```bash
# Create parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name my-params \
  --db-parameter-group-family postgres14 \
  --description "Custom parameters"

# Modify parameter
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-params \
  --parameters "ParameterName=max_connections,ParameterValue=500,ApplyMethod=immediate"

# Apply to instance
aws rds modify-db-instance \
  --db-instance-identifier my-database \
  --db-parameter-group-name my-params
```

### DynamoDB Fundamentals

#### 1. Provisioned vs On-Demand

**Provisioned Capacity:**
- Specify RCU (Read Capacity Units) and WCU (Write Capacity Units)
- Cost: Fixed per RCU/WCU per hour
- Autoscaling available
- Better for predictable traffic

**On-Demand Capacity:**
- Pay per request
- No capacity planning
- Suitable for unpredictable traffic
- More expensive at high volume

**RCU/WCU Calculation:**
- 1 RCU = 1 strongly consistent read (4 KB item)
- 1 WCU = 1 write (1 KB item)
- Eventually consistent reads = 0.5 RCU

```bash
# Provisioned table
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=userId,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5

# On-demand table
aws dynamodb create-table \
  --table-name Events \
  --attribute-definitions AttributeName=eventId,AttributeType=S \
  --key-schema AttributeName=eventId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

#### 2. Global Tables (Multi-Region)

**Features:**
- Active-active replication
- Sub-second replication
- Automatic conflict resolution (last-write-wins)
- All regions read/write capable

```bash
# Create global table
aws dynamodb create-global-table \
  --global-table-name Users \
  --replication-group RegionName=us-east-1 RegionName=eu-west-1

# Add region
aws dynamodb update-table \
  --table-name Users \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --replicas "RegionName=ap-southeast-1"
```

---

## 1.5 Networking: VPC, Route53, CloudFront

### VPC Architecture

#### 1. VPC Components

**VPC:**
- Isolated network in AWS account
- CIDR block defines IP range (e.g., 10.0.0.0/16)
- Can have public and private subnets

**Subnets:**
- Partition of VPC CIDR block
- Within single AZ
- Public: Routes traffic to IGW
- Private: No direct internet access

**Route Tables:**
- Define routing rules for subnets
- Routes to IGW, NAT, peering, VPN
- Default route (0.0.0.0/0) for unmatched traffic

**Internet Gateway (IGW):**
- Enables communication between VPC and internet
- Managed by AWS
- Supports both IPv4 and IPv6

**NAT Gateway:**
- Allows private subnets to access internet
- Must be in public subnet
- Translates private IPs to EIP
- Stateful (return traffic allowed)
- High availability per AZ

**VPC Flow Logs:**
- Capture network traffic metadata
- Can log to CloudWatch or S3
- Troubleshoot connectivity issues
- Security monitoring

```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create public subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a

# Create private subnet
aws ec2 create-subnet \
  --vpc-id vpc-12345 \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1a

# Create internet gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-12345 \
  --vpc-id vpc-12345

# Create route to IGW
aws ec2 create-route \
  --route-table-id rtb-12345 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-12345
```

#### 2. NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|---|---|
| **Type** | AWS managed | EC2 instance |
| **Throughput** | 100 Gbps | Limited to instance size |
| **Availability** | Auto HA per AZ | Manual HA setup |
| **Cost** | Per GB processed | Instance + data transfer |
| **Maintenance** | None | Patches, updates needed |
| **Performance** | Optimized | Variable |

**Recommendation:** Use NAT Gateway (managed service preferred)

#### 3. VPC Endpoints

**Gateway Endpoints (S3, DynamoDB):**
- Free
- Route table entry
- Enables private access
- No data charges

**Interface Endpoints (other services):**
- Powered by PrivateLink
- Cost: Per endpoint per hour
- Multiple security groups possible

```bash
# S3 Gateway Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-12345

# Lambda Interface Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.lambda \
  --subnet-ids subnet-12345
```

#### 4. Security Groups vs NACLs

**Security Groups Evaluation:**
```
1. Check source IP/security group in inbound rule
2. If match, check destination port and protocol
3. If rule matched, allow traffic
4. Default deny
```

**NACL Evaluation Order:**
```
1. Evaluate rules in order (lowest number first)
2. First matching rule (allow or deny) wins
3. If no match, default allow (for default NACL)
4. Rules evaluated in both directions (stateless)
```

**Troubleshooting Connectivity:**
- Check security group rules (both inbound/outbound)
- Check NACLs for both source and destination subnets
- Check route tables (packets finding path)
- Check NAT configuration if private subnet

### Route53 Fundamentals

#### 1. Routing Policies

**Simple Routing:**
- Single resource
- No health checks
- Use for simple DNS needs

**Weighted Routing:**
- Distribute traffic by percentage
- Use cases: A/B testing, canary deployments

```bash
# 90% to version 1, 10% to version 2
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "v1",
          "Weight": 90,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "v1-alb.example.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.example.com",
          "Type": "A",
          "SetIdentifier": "v2",
          "Weight": 10,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "v2-alb.example.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

**Latency-Based Routing:**
- Route to region with lowest latency
- Use cases: Global applications
- Route53 measures latency to each region

**Failover Routing:**
- Primary + secondary resources
- Health checks determine active
- Automatic failover

**Geolocation Routing:**
- Route based on user location
- Use cases: Compliance, localized content
- By country, continent, or default

**Geoproximity Routing:**
- Route based on geographic proximity + bias
- Bias: Shift boundary toward/away region
- Use cases: Proximity-based load balancing

**Multi-Value Routing:**
- Multiple healthy targets
- Randomly returns up to 8 IPs
- Similar to simple but with health checks
- Not true load balancing (use ELB instead)

#### 2. Health Checks

**Endpoint Health Checks:**
- HTTP(S) or TCP connection
- Can verify response content
- CloudWatch Alarm based

```bash
aws route53 create-health-check \
  --health-check-config '{
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "example.com",
    "Port": 443,
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'
```

**CloudWatch Alarm Health Checks:**
- Link to CloudWatch alarm
- Custom logic in alarm

**Calculated Health Checks:**
- Combine multiple health checks
- Custom logic

### CloudFront Fundamentals

#### 1. Distribution Setup

**Origins:**
- S3 bucket, ELB, EC2, API Gateway
- Distributes content geographically
- Edge locations cache content

**Behaviors:**
- Path patterns trigger different cache behaviors
- Different origins for different paths

```bash
# Create distribution
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "unique-id",
    "Comment": "My distribution",
    "DefaultCacheBehavior": {
      "TargetOriginId": "my-alb",
      "ViewerProtocolPolicy": "redirect-to-https",
      "TrustedSigners": {
        "Enabled": false,
        "Quantity": 0
      },
      "ForwardedValues": {
        "QueryString": true,
        "Cookies": { "Forward": "all" },
        "Headers": {
          "Quantity": 0
        }
      },
      "MinTTL": 0,
      "DefaultTTL": 86400,
      "MaxTTL": 31536000
    },
    "Origins": {
      "Items": [
        {
          "Id": "my-alb",
          "DomainName": "my-alb.elb.amazonaws.com",
          "CustomOriginConfig": {
            "HTTPPort": 80,
            "HTTPSPort": 443,
            "OriginProtocolPolicy": "https-only"
          }
        }
      ],
      "Quantity": 1
    },
    "Enabled": true
  }'
```

#### 2. Caching & Invalidation

**TTL (Time To Live):**
- Default: 86400 seconds (24 hrs)
- Objects cached at edge locations
- After TTL expires, fetch from origin

**Cache Keys:**
- By default: Full URL
- Can include query strings, headers, cookies
- Affects cache hit ratio

**Invalidation:**
- Force cache refresh before TTL expiration
- Use paths: `/`, `/*`, `/images/*`
- Charges apply for invalidations beyond free tier

```bash
# Invalidate cache
aws cloudfront create-invalidation \
  --distribution-id E123ABC \
  --paths "/*"
```

---

## Phase 1 Interview Practice Questions

### Quick Fire Questions (5 minutes each)

1. **IAM:** What happens if both Deny and Allow policies exist?
   - Answer: Explicit Deny always wins (Default Deny Logic)

2. **EC2:** What is the difference between a Security Group and a NACL?
   - Answer: SG is stateful, instance-level; NACL is stateless, subnet-level

3. **S3:** Can you make a private S3 bucket temporarily public?
   - Answer: Yes, via bucket policy or ACL, but risks accidental exposure

4. **RDS:** What's the difference between Multi-AZ and Read Replicas?
   - Answer: Multi-AZ is sync failover same region; Replicas are async, any region

5. **VPC:** How does a NAT Gateway work?
   - Answer: Private instance sends to NAT, NAT changes source IP to its EIP

### Scenario Questions

**Scenario 1: Microservices on EC2**
- Multiple services need S3 access, each to different bucket
- How would you manage IAM permissions?
- Answer: Create IAM roles per service, attach to EC2 instances, use resource ARNs in policies

**Scenario 2: Database Disaster Recovery**
- RPO: 1 hour, RTO: 4 hours
- How would you design with RDS?
- Answer: Multi-AZ primary, cross-region read replica, automated backups, promote replica on failure

**Scenario 3: Global Content Distribution**
- Static website needs to be served globally with low latency
- Architecture?
- Answer: S3 + CloudFront with multiple edge locations, Route53 geolocation routing

---

## Study Tips & Resources

### Key Concepts to Master
1. **IAM:** Trust relationships, cross-account access, least privilege
2. **EC2:** Instance types, scaling, load balancing, health checks
3. **S3:** Bucket policies, encryption, lifecycle, versioning
4. **RDS:** Multi-AZ, read replicas, backups, RPO/RTO
5. **VPC:** Subnets, routing, security, NAT

### Common Mistakes to Avoid
- Using IAM Users instead of Roles for services
- Forgetting security group rules for load balancer health checks
- Not enabling versioning before replication
- Single-AZ for production databases
- Overly permissive IAM policies

### Practice Projects
1. Create VPC with public/private subnets and NAT
2. Set up RDS Multi-AZ with read replica
3. Deploy application on EC2 with ALB and ASG
4. Create S3 bucket with lifecycle and encryption
5. Implement cross-account access pattern

---

**Total Phase 1 Study Time: 4-6 hours**
**Hands-on Labs Time: 3-4 hours**
