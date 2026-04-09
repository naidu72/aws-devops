# AWS EKS - Architecture Deep Dive

## Module Overview

**Learning Objectives:**
- Understand EKS control plane architecture
- Master node group management and scaling
- Learn IAM integration and IRSA (IAM Roles for Service Accounts)
- Deep dive managed add-ons
- VPC networking integration
- Design production EKS clusters

**Estimated Study Time:** 10-12 hours
**Real-world Context:** Workspace uses EKS extensively (see `hansen-cloud-native-suite/eksv2/`)

---

## Part 1: EKS Control Plane

### 1.1 What AWS Manages vs What You Manage

**AWS Manages (Control Plane):**
```
┌────────────────────────────────────────────┐
│          AWS EKS (Managed)                 │
├────────────────────────────────────────────┤
│ • kube-apiserver (3+ replicas, multi-AZ)   │
│ • etcd (encrypted at rest, multi-AZ)       │
│ • kube-controller-manager (HA)             │
│ • kube-scheduler (HA)                      │
│ • Auto-patching & upgrades                 │
│ • 99.95% SLA (multi-AZ)                    │
└────────────────────────────────────────────┘
```

**You Manage (Data Plane):**
```
┌────────────────────────────────────────────┐
│          Worker Nodes (EC2)                │
├────────────────────────────────────────────┤
│ • kubelet                                  │
│ • Container runtime (docker, containerd)   │
│ • Pod networking (CNI plugin)              │
│ • Node patching & OS updates               │
│ • Application workloads                    │
└────────────────────────────────────────────┘
```

### 1.2 EKS Control Plane Architecture

**Multi-AZ Deployment:**
```
┌─────────────────────────────────────────────────────────┐
│                    AWS Account                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  AZ-1              AZ-2              AZ-3              │
│  ┌───────┐       ┌───────┐       ┌───────┐            │
│  │ API   │       │ API   │       │ API   │            │
│  │Server │       │Server │       │Server │            │
│  └───────┘       └───────┘       └───────┘            │
│     │               │               │                  │
│  ┌──────────────────────────────────────┐             │
│  │      etcd Cluster (3 members)        │             │
│  │  Replicated, consistent, HA          │             │
│  └──────────────────────────────────────┘             │
│     │               │               │                  │
│  ┌──────────────────────────────────────┐             │
│  │  Controller Manager & Scheduler      │             │
│  │  (Run on each AZ, coordinated)       │             │
│  └──────────────────────────────────────┘             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Key Features:**
- **Multi-AZ:** Survives single AZ failure
- **Auto-patching:** OS and component updates without downtime
- **Encryption at rest:** etcd encrypted with AWS KMS (if configured)
- **API Server Endpoints:** Can be public (internet) or private (VPN only)

### 1.3 Cluster Endpoint Configuration

```bash
# Public endpoint (default)
# - Accessible from internet (with AWS credentials)
# - Used by developers, CI/CD

# Private endpoint
# - Accessible only within VPC
# - Used in highly secure environments
# - Requires VPN or bastion host

# Both endpoints (recommended)
# - Public for outside access
# - Private for VPC-internal access
```

**Terraform Configuration (from workspace):**
```hcl
resource "aws_eks_cluster" "main" {
  name            = "my-cluster"
  version         = "1.28"
  role_arn        = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids              = var.subnet_ids  # Public & private
    endpoint_private_access = true            # VPC access
    endpoint_public_access  = true            # Internet access
    public_access_cidrs    = ["0.0.0.0/0"]    # Who can access API
  }

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]  # etcd encryption
  }

  tags = {
    Environment = "prod"
  }
}
```

### 1.4 Control Plane Logs

**Types Available:**
```
1. API Server (API requests, authentication)
2. Audit (who did what, when - compliance)
3. Authenticator (authentication debugging)
4. Controller Manager (deployment reconciliation)
5. Scheduler (pod scheduling decisions)
```

**Enabling Logs to CloudWatch:**
```hcl
enabled_cluster_log_types = ["api", "audit", "scheduler"]
```

**Query Examples:**
```bash
# Find API errors
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/cluster \
  --filter-pattern "[ERROR]"

# Find failed authentications
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/authenticator \
  --filter-pattern "error"

# Audit log (who deleted pods)
aws logs filter-log-events \
  --log-group-name /aws/eks/my-cluster/audit \
  --filter-pattern "\"verb\":\"delete\""
```

**Interview Question:** "How would you troubleshoot an authentication issue in EKS?"
**Answer:**
- Check authenticator logs (CloudWatch)
- Verify IAM role/user has `eks:DescribeCluster` permission
- Check kubeconfig entry matches cluster ARN
- Test: `aws sts get-caller-identity` (shows authenticated principal)
- If RBAC error, check `kubectl describe clusterrolebinding`

---

## Part 2: Node Groups

### 2.1 Node Group Types

**Managed Node Groups (AWS manages nodes):**
- Auto Scaling Group created/managed by AWS
- Updates handled by AWS
- Recommended for most use cases
- Pay for EC2 instances only

**Self-Managed Node Groups (You manage nodes):**
- Traditional Auto Scaling Group
- You handle AMI, patches, updates
- More control but more operational burden
- Used for specialized scenarios (GPU, custom kernels)

**Fargate (Serverless pods):**
- AWS manages compute
- No nodes to manage
- Pay per vCPU-hour
- Good for batch, variable workloads

### 2.2 Managed Node Group Configuration

**Terraform Example (from workspace):**
```hcl
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "default"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.subnet_ids

  # Scaling configuration
  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 1
  }

  # Instance types & capacity type
  instance_types = ["t3.large", "t3a.large"]  # Multiple types for flexibility
  capacity_type  = "SPOT"                     # ON_DEMAND or SPOT (cost saving)

  # Kubernetes labels & taints
  labels = {
    Environment = "prod"
    WorkloadType = "general"
  }

  taints {
    key    = "dedicated"
    value  = "workload1"
    effect = "NO_SCHEDULE"
  }

  # Launch template for customization
  launch_template {
    id      = aws_launch_template.nodes.id
    version = "$Latest"
  }

  tags = {
    Name = "default-node-group"
  }

  depends_on = [aws_iam_role_policy.node_AmazonEKSWorkerNodePolicy]
}

# Launch Template for node customization
resource "aws_launch_template" "nodes" {
  name_prefix = "eks-node-"

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 100      # Root volume size
      volume_type           = "gp3"    # General Purpose SSD
      encrypted             = true
      delete_on_termination = true
    }
  }

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 only (security)
    http_put_response_hop_limit = 1
  }

  monitoring {
    enabled = true  # CloudWatch detailed monitoring
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "eks-node"
    }
  }
}
```

### 2.3 Node Group Scaling

**Manual Scaling:**
```bash
# Update desired_size in Terraform
terraform apply

# Or via AWS CLI
aws eks update-nodegroup-config \
  --cluster-name my-cluster \
  --nodegroup-name default \
  --scaling-config minSize=1,maxSize=20,desiredSize=5
```

**Auto Scaling (Cluster Autoscaler):**
```yaml
# Installed by workspace in eksConfig/scaling/clusterAutoscaler/
# Watches for pending pods
# If pod pending due to insufficient resources:
# 1. Scale up the node group
# 2. Retry pod scheduling
# If nodes idle for 10 min:
# 3. Scale down the node group
```

**Karpenter (Advanced Autoscaling):**
```yaml
# Workspace also uses Karpenter for faster scaling
# Advantages over Cluster Autoscaler:
# - Faster scaling (seconds vs minutes)
# - Bin-packing (consolidate pods, reduce nodes)
# - Native Spot instance support
# - Provisionless (no auto-scaling groups)
```

### 2.4 Node Group Upgrade Process

```
1. Mark node group for update:
   aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name default --kubernetes-version 1.28

2. AWS cordons nodes (no new pods scheduled)

3. Existing pods evicted (respects PodDisruptionBudgets)

4. New AMI launched with updated K8s version

5. Process repeats for each node in group

Zero-downtime if:
- Multiple replicas of workloads
- PodDisruptionBudgets defined
- Sufficient cluster capacity for pod redistribution
```

---

## Part 3: IAM Integration & IRSA

### 3.1 Traditional Pod Authentication (Legacy)

**Problem:** Pods need AWS permissions (S3, SQS, etc.)
- Store AWS credentials as Kubernetes Secret
- Security risk: credentials shared across pods
- Hard to rotate, audit, least-privilege

### 3.2 IRSA: IAM Roles for Service Accounts (Modern)

**Solution:** Pod authenticates via ServiceAccount, trusts IAM role

**Architecture:**
```
Pod (with ServiceAccount)
    ↓
  (Requests AWS credentials)
    ↓
IRSA Webhook (mutates pod)
    ↓
  (Injects OIDC token)
    ↓
Pod makes AWS API call (includes OIDC token)
    ↓
AWS IAM (validates token, trusts OIDC provider)
    ↓
Issue temporary credentials
    ↓
Pod gets S3 access, SQS access, etc.
```

### 3.3 IRSA Setup (Step by Step)

**Step 1: Enable OIDC Provider**
```hcl
resource "aws_eks_cluster" "main" {
  # ... cluster config
}

# Get OIDC provider thumbprint
data "aws_eks_cluster_auth" "main" {
  name = aws_eks_cluster.main.name
}

# Create OIDC identity provider
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["9e99a48a9960b14926bb7f3b02e22da2b0ab7280"]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
}
```

**Step 2: Create IAM Role**
```hcl
resource "aws_iam_role" "s3_access" {
  name = "eks-pod-s3-access"

  # Trust the OIDC provider
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:default:my-app"
          }
        }
      }
    ]
  })
}

# Attach S3 permissions
resource "aws_iam_role_policy" "s3_access" {
  name = "s3-access"
  role = aws_iam_role.s3_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::my-bucket",
          "arn:aws:s3:::my-bucket/*"
        ]
      }
    ]
  })
}
```

**Step 3: Annotate ServiceAccount**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/eks-pod-s3-access
```

**Step 4: Pod Automatically Gets Credentials**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s3-reader
spec:
  serviceAccountName: my-app  # Uses annotated ServiceAccount
  containers:
  - name: app
    image: myapp:latest
    # AWS SDK automatically uses IRSA credentials
    # No need to pass AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
```

**How it Works:**
```bash
# Inside pod:
$ aws s3 ls
# Works! No credentials in container

# SDK checks environment (in order):
# 1. AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
# 2. EKS IRSA (OIDC token → IAM role)
# 3. IAM instance profile (node role)

# IRSA provides short-lived credentials (1 hour max)
# Automatic refresh before expiry
```

### 3.4 IRSA Best Practices

**Principle of Least Privilege:**
```hcl
# Bad: Admin permissions
{
  Effect = "Allow"
  Action = "s3:*"
  Resource = "*"
}

# Good: Only needed permissions
{
  Effect = "Allow"
  Action = [
    "s3:GetObject",
    "s3:ListBucket"
  ]
  Resource = [
    "arn:aws:s3:::my-app-bucket",
    "arn:aws:s3:::my-app-bucket/*"
  ]
}
```

**Separate Roles per App:**
```
App1 → IAM Role with S3 only
App2 → IAM Role with DynamoDB only
App3 → IAM Role with SQS only

(Not: One role with all permissions)
```

---

## Part 4: Managed Add-ons

AWS manages specific Kubernetes components as add-ons:

### 4.1 VPC CNI Add-on

**What it does:** Pod networking (IP assignment)

**Configuration:**
```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "vpc-cni"
  addon_version            = "v1.14.1-eksbuild.1"
  resolve_conflicts_on_create = "OVERWRITE"

  # Optional: IRSA for CNI pod
  service_account_role_arn = aws_iam_role.vpc_cni.arn
}
```

**Key Parameters:**
```yaml
# env.aws-node (DaemonSet):
- name: WARM_IP_TARGET
  value: "5"  # Maintain 5 warm IPs per node

- name: MINIMUM_IP_TARGET
  value: "10" # Keep at least 10 IPs per node

- name: MAXIMUM_IP_TARGET
  value: "20" # Maximum 20 IPs per node

- name: AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG
  value: "false"  # true for custom network config (advanced)

- name: POD_SECURITY_GROUP_ENFORCING_MODE
  value: "standard"  # standard or strict
```

**Interview Question:** "What happens when VPC runs out of IP addresses on a node?"
**Answer:**
- New pods fail to schedule (no IP available)
- Warning: "Failed to allocate IP for pod"
- Solutions:
  1. Larger VPC CIDR (e.g., /16 instead of /24)
  2. Enable prefix delegation (assigns /28 to ENI, not individual IPs)
  3. Switch to Calico CNI (doesn't use VPC IPs)

### 4.2 kube-proxy Add-on

**What it does:** Service networking (iptables/ipvs rules)

```hcl
resource "aws_eks_addon" "kube_proxy" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "kube-proxy"
  addon_version = "v1.28.1-eksbuild.1"
}
```

### 4.3 CoreDNS Add-on

**What it does:** DNS for service discovery

```hcl
resource "aws_eks_addon" "coredns" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "coredns"
  addon_version = "v1.10.1-eksbuild.2"
}
```

### 4.4 EBS CSI Driver Add-on

**What it does:** Attach EBS volumes to pods (StatefulSet persistence)

```hcl
resource "aws_eks_addon" "ebs_csi_driver" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "aws-ebs-csi-driver"
  addon_version            = "v1.25.0-eksbuild.1"
  service_account_role_arn = aws_iam_role.ebs_csi.arn
}

resource "aws_iam_role" "ebs_csi" {
  name = "eks-ebs-csi-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.eks.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:kube-system:ebs-csi-controller-sa"
          }
        }
      }
    ]
  })
}

# Attach AWS-managed policy
resource "aws_iam_role_policy_attachment" "ebs_csi" {
  role       = aws_iam_role.ebs_csi.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}
```

---

## Part 5: VPC Integration

### 5.1 VPC Architecture for EKS

```
┌──────────────────────────────────────────────┐
│           AWS VPC (10.0.0.0/16)              │
├──────────────────────────────────────────────┤
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │  Public Subnet (10.0.1.0/24)        │    │
│  │  - NAT Gateway                      │    │
│  │  - EKS Control Plane (public only)  │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │  Private Subnet 1 (10.0.10.0/24)    │    │
│  │  - EKS Nodes (AZ-1)                 │    │
│  │  - Pods (10.1.0.0/24 - secondary)   │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │  Private Subnet 2 (10.0.11.0/24)    │    │
│  │  - EKS Nodes (AZ-2)                 │    │
│  │  - Pods (10.2.0.0/24 - secondary)   │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │  Private Subnet 3 (10.0.12.0/24)    │    │
│  │  - EKS Nodes (AZ-3)                 │    │
│  │  - Pods (10.3.0.0/24 - secondary)   │    │
│  └─────────────────────────────────────┘    │
│                                              │
└──────────────────────────────────────────────┘
```

### 5.2 Secondary CIDR for Pod Networking

**Why:** VPC CIDR (10.0.0.0/16) limited; pods need own IP range

**Configuration:**
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# Add secondary CIDR for pods
resource "aws_vpc_ipv4_cidr_block_association" "pods" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "100.64.0.0/16"  # Secondary CIDR
}

# Private subnets with secondary CIDR
resource "aws_subnet" "private_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"  # Node subnet
}

# Pod subnet (from secondary CIDR)
resource "aws_subnet" "pods_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "100.64.1.0/24" # Pod subnet
  availability_zone = "us-east-1a"
}
```

**Amazon VPC CNI uses both:**
- Nodes get IPs from VPC CIDR
- Pods get IPs from secondary CIDR

### 5.3 Security Groups for Pods

**Default:** All pods on node share host security group

**Advanced:** Pod-level security groups
```hcl
# Enable pod security groups
vpc_config {
  enable_pod_security_group = true
}
```

**Usage:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  annotations:
    vpc.amazonaws.com/eni-security-groups: sg-0123456789abcdef0
spec:
  containers:
  - name: app
    image: app:latest
```

**Benefit:** Different pods on same node, different security group rules

---

## Part 6: Production EKS Checklist

### Essential Setup:
- [ ] Multi-AZ cluster (at least 3 AZs)
- [ ] OIDC provider configured
- [ ] IRSA roles for workloads
- [ ] Managed add-ons enabled & updated
- [ ] VPC CNI optimized (WARM_IP_TARGET tuned)
- [ ] Control plane logs to CloudWatch
- [ ] etcd encrypted with KMS
- [ ] IMDSv2 enforced (metadata security)
- [ ] Auto Scaling Group or Karpenter configured
- [ ] Pod Disruption Budgets for workloads

### Networking:
- [ ] Public & private subnets across 3 AZs
- [ ] NAT Gateway for egress (high availability)
- [ ] Security groups configured per workload
- [ ] Network policies implemented (zero-trust)
- [ ] Ingress controller deployed (ingress-nginx or ALB)

### Observability:
- [ ] CloudWatch Container Insights enabled
- [ ] Prometheus + Grafana for metrics
- [ ] Centralized logging (ELK, Loki, CloudWatch)
- [ ] Distributed tracing (Jaeger, X-Ray)
- [ ] Alert rules configured

### Security:
- [ ] Pod Security Policies or Pod Security Standards
- [ ] RBAC roles & bindings (least privilege)
- [ ] Secrets encryption (AWS Secrets Manager or Vault)
- [ ] Image scanning in ECR
- [ ] Network policies (deny-all default)
- [ ] Regular security audits (kube-bench)

---

## Part 7: Interview Scenarios

### Scenario 1: Design Production EKS Cluster

**Requirements:**
- 3 microservices (stateless)
- 1 PostgreSQL database (stateful)
- Expected load: 1000 RPS
- Budget-conscious startup

**Your Answer:**
1. **Cluster:**
   - Multi-AZ (3 nodes minimum, 1 per AZ)
   - Managed node group with auto-scaling
   - On-demand instances (or Spot for non-critical)

2. **Networking:**
   - VPC with private subnets for nodes
   - Secondary CIDR for pod networking
   - ALB ingress for north-south traffic
   - Security groups per tier

3. **Data Persistence:**
   - RDS for PostgreSQL (managed, HA)
   - EBS CSI driver for PVCs (if needed)
   - Automated backups

4. **Identity & Access:**
   - IRSA for pod identity (S3, SQS access)
   - RBAC for human users
   - ServiceAccounts per microservice

5. **Observability:**
   - CloudWatch Container Insights
   - Prometheus + Grafana
   - Centralized logging
   - Alarms for critical metrics

6. **Cost Optimization:**
   - Spot instances for non-critical workloads
   - Right-sizing instances (t3.medium, not t3.2xlarge)
   - Cluster autoscaler (scale down unused nodes)

---

### Scenario 2: Troubleshoot Pod Can't Access S3

**Problem:** Pod deployed, tries to access S3, gets "403 Forbidden"

**Diagnosis:**
1. Check ServiceAccount annotation:
   ```bash
   kubectl describe sa <pod-sa> -n <namespace>
   # Should have eks.amazonaws.com/role-arn annotation
   ```

2. Check IAM role trust policy:
   ```bash
   aws iam get-role --role-name <role-name>
   # Should trust the OIDC provider
   ```

3. Check IAM role permissions:
   ```bash
   aws iam list-role-policies --role-name <role-name>
   aws iam get-role-policy --role-name <role-name> --policy-name <policy>
   # Should have s3:GetObject on bucket
   ```

4. Check pod has OIDC token:
   ```bash
   kubectl exec <pod> -- env | grep AWS
   # Should have AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE
   ```

5. Test AWS SDK:
   ```bash
   kubectl exec <pod> -- aws s3 ls
   # Should list buckets or fail with clear error
   ```

---

## Part 8: Key Takeaways

1. **Control Plane:** AWS manages (SLA 99.95%), multi-AZ by default
2. **Node Groups:** Managed (recommended) or self-managed
3. **IRSA:** Modern way to grant AWS permissions (not credentials in Secret)
4. **Add-ons:** VPC CNI, kube-proxy, CoreDNS managed by AWS
5. **Scaling:** Cluster Autoscaler or Karpenter
6. **VPC Integration:** Multi-AZ, secondary CIDR, security groups
7. **Security:** OIDC, RBAC, NetworkPolicies, encryption

---

## Study Plan (Next 2-3 Hours)

1. **Theory (45 min):** Control plane, node groups, IRSA
2. **Configuration (45 min):** Review workspace EKS Terraform
3. **Troubleshooting (30 min):** Practice common issues
4. **Design Exercise (30 min):** Design EKS cluster for new startup
5. **Interview Prep (15 min):** Practice Scenario 1 & 2

**Next Module:** Learn AKS (Azure Kubernetes Service)
