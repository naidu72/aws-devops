# 01 - EKS Basics: Architecture and Multi-Cluster Setup

## What is Amazon EKS?

Amazon EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes service. AWS manages the Kubernetes control plane, and you manage the worker nodes. This simplifies cluster operations while giving you full Kubernetes capabilities.

### EKS vs Self-Managed Kubernetes

| Aspect | EKS | Self-Managed |
|--------|-----|--------------|
| Control Plane | AWS managed | You manage |
| Updates | Automatic | Manual |
| HA | Built-in (3 AZs) | You implement |
| Monitoring | CloudWatch integrated | You implement |
| Support | AWS support | Community |
| Cost | Higher per cluster | Lower but higher ops |
| Reliability | 99.95% SLA | Your responsibility |

---

## EKS Architecture

### Control Plane Components (AWS Managed)
```
┌─────────────────────────────────────────────────┐
│  AWS EKS Control Plane (Multi-AZ)              │
│  ┌────────────────────────────────────────┐    │
│  │ API Server (endpoint)                  │    │
│  │ etcd (managed backup & restore)        │    │
│  │ Scheduler                              │    │
│  │ Controller Manager                     │    │
│  └────────────────────────────────────────┘    │
│  Span 3 Availability Zones                     │
│  Auto-update & patch management                │
└─────────────────────────────────────────────────┘
          ↓ (Kubernetes API)
      [Your VPC]
```

### Data Plane (You Manage)
```
┌─────────────────────────────────────────────────┐
│  Your VPC & Subnets                            │
│  ┌─────────────┐  ┌─────────────┐             │
│  │  AZ-1 Node  │  │  AZ-2 Node  │             │
│  │  ┌────────┐ │  │  ┌────────┐ │             │
│  │  │ Pod    │ │  │  │ Pod    │ │             │
│  │  │ Pod    │ │  │  │ Pod    │ │             │
│  │  │ Pod    │ │  │  │ Pod    │ │             │
│  │  └────────┘ │  │  └────────┘ │             │
│  │  kubelet    │  │  kubelet    │             │
│  │  kube-proxy │  │  kube-proxy │             │
│  └─────────────┘  └─────────────┘             │
│  ┌─────────────┐                              │
│  │  AZ-3 Node  │                              │
│  │  ┌────────┐ │                              │
│  │  │ Pod    │ │                              │
│  │  │ Pod    │ │                              │
│  │  └────────┘ │                              │
│  │  kubelet    │                              │
│  │  kube-proxy │                              │
│  └─────────────┘                              │
│  Multi-AZ for high availability               │
│  Auto-Scaling Groups manage node lifecycle    │
└─────────────────────────────────────────────────┘
```

---

## Your Multi-Cluster Architecture

### DEV, QA, Staging, Production Setup

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Account                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │    DEV       │  │     QA       │  │   Staging    │     │
│  │  EKS Cluster │  │ EKS Cluster  │  │ EKS Cluster  │     │
│  │              │  │              │  │              │     │
│  │ Control Plane│  │ Control Plane│  │ Control Plane│     │
│  │ 1-2 Nodes    │  │ 2-3 Nodes    │  │ 2-3 Nodes    │     │
│  │ t3.medium    │  │ t3.large     │  │ t3.large     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Production EKS Cluster (HA)               │    │
│  │  ┌──────────────────────────────────────────┐     │    │
│  │  │ Control Plane (Multi-AZ)                │     │    │
│  │  │ etcd backed by AWS                      │     │    │
│  │  └──────────────────────────────────────────┘     │    │
│  │  ┌──────────────────────────────────────────┐     │    │
│  │  │ Data Plane: 6-12 Nodes (c5.xlarge)      │     │    │
│  │  │ Spread across 3 AZs                     │     │    │
│  │  │ Auto-Scaling Group: 6-12 nodes          │     │    │
│  │  └──────────────────────────────────────────┘     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Cluster Specifications

| Aspect | DEV | QA | Staging | Production |
|--------|-----|-----|---------|------------|
| Nodes | 1-2 | 2-3 | 2-3 | 6-12 |
| Node Type | t3.medium | t3.large | t3.large | c5.xlarge |
| AZs | 1-2 | 2 | 3 | 3 |
| Control Plane | Shared | Shared | Shared | Dedicated |
| Namespaces | 3-5 | 5-8 | 5-8 | 8-10 |
| Pod Limits | 100-150 | 150-300 | 200-400 | 500+ |
| Security Groups | Permissive | Restrictive | Restrictive | Highly Restrictive |

---

## EKS Add-ons (Critical Components)

EKS manages these add-ons for you:

### 1. **VPC CNI (vpc-cni)**
```yaml
# Manages pod networking
# Assigns ENIs (Elastic Network Interfaces) to nodes
# Each pod gets an IP from your VPC subnet
# Enable: aws eks list-addons --cluster-name my-cluster
```

**Why it matters:**
- Pods have real VPC IPs (not overlay network)
- Enables direct connectivity to AWS services
- No performance overhead of overlay networks

### 2. **CoreDNS**
```yaml
# Kubernetes DNS server
# Enables service discovery (service-name.namespace.svc.cluster.local)
# Deployed as Deployment in kube-system namespace
```

**Why it matters:**
- Services are discovered via DNS
- Applications don't need to know pod IPs
- Automatic load balancing via DNS

### 3. **kube-proxy**
```yaml
# Network plugin that implements Services
# Creates iptables rules for service routing
# Runs on every node as DaemonSet
```

**Why it matters:**
- Services route traffic to pods
- Handles ClusterIP, NodePort, LoadBalancer types
- Manages connection load balancing

### 4. **EC2 IMDS v2** (Instance Metadata Service)
```yaml
# Security best practice
# Tokens required to access instance metadata
# Prevents SSRF attacks from containers
```

---

## EKS Cluster Networking

### VPC Integration

```
┌─────────────────────────────────────────┐
│         VPC (e.g., 10.0.0.0/16)        │
│ ┌──────────────────────────────────┐   │
│ │    Public Subnets (AZ-1)        │   │
│ │    10.0.1.0/24 (NAT GW)         │   │
│ │                                 │   │
│ │ ┌────────────────────────────┐  │   │
│ │ │  Node 1 (t3.large)        │  │   │
│ │ │  10.0.1.100               │  │   │
│ │ │  ┌──────────────────────┐ │  │   │
│ │ │  │ Pod: 10.0.1.101     │ │  │   │
│ │ │  │ Pod: 10.0.1.102     │ │  │   │
│ │ │  └──────────────────────┘ │  │   │
│ │ └────────────────────────────┘  │   │
│ └──────────────────────────────────┘   │
│                                        │
│ ┌──────────────────────────────────┐   │
│ │    Private Subnets (AZ-1)       │   │
│ │    10.0.10.0/24 (NAT GW)        │   │
│ │    No direct internet access    │   │
│ └──────────────────────────────────┘   │
│                                        │
│ Control Plane Endpoint:               │
│ https://XXXXX.eks.amazonaws.com     │
│ (Uses VPC Endpoint for private)      │
└─────────────────────────────────────────┘
```

### Network Flow

```
Pod (10.0.1.101)
    ↓ (wants to reach another pod)
kubelet detects outgoing traffic
    ↓
kube-proxy checks iptables rules
    ↓
Routes to destination pod on same/different node
    ↓
Destination pod (10.0.2.105)
```

---

## Node Groups and Auto-Scaling

### Node Group Structure

```
┌──────────────────────────────────────────┐
│    EKS Cluster "my-prod-cluster"        │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │   Node Group: "on-demand"       │   │
│  │   • Launch template: EKS-opt AMI│   │
│  │   • Instance type: c5.xlarge    │   │
│  │   • Capacity: On-Demand         │   │
│  │   • Min: 3, Desired: 6, Max: 12 │   │
│  │   • ASG: Auto-scaling group     │   │
│  │                                 │   │
│  │   ┌─────────────────────────┐   │   │
│  │   │ Node-1 (AZ-1)          │   │   │
│  │   │ Node-2 (AZ-2)          │   │   │
│  │   │ Node-3 (AZ-3)          │   │   │
│  │   │ Node-4 (AZ-1)          │   │   │
│  │   └─────────────────────────┘   │   │
│  └──────────────────────────────────┘   │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │   Node Group: "spot"             │   │
│  │   • Instance type: c5.xlarge     │   │
│  │   • Capacity: Spot (savings)     │   │
│  │   • Min: 0, Desired: 3, Max: 10  │   │
│  │   • ASG: For batch/non-critical  │   │
│  └──────────────────────────────────┘   │
│                                         │
└──────────────────────────────────────────┘
```

### How Auto-Scaling Works

```
1. Pod pending (not enough resources)
   ↓
2. Kubernetes scheduler can't place pod
   ↓
3. Cluster Autoscaler detected pending pod
   ↓
4. Autoscaler scales up Auto-Scaling Group
   ↓
5. New node launched (EC2 instance)
   ↓
6. Node joins cluster (runs kubelet)
   ↓
7. Pod scheduled to new node
   ↓
8. Application running!

Time to scale up: 1-3 minutes
Time to scale down (idle nodes): 10 minutes
```

---

## Cluster Communication Flow

### External → Cluster

```
User: kubectl get pods
        ↓
kubectl reads kubeconfig
        ↓
kubeconfig points to: https://XXXXX.eks.amazonaws.com
        ↓
AWS SigV4 signs request with IAM credentials
        ↓
EKS API Server endpoint
        ↓
kube-apiserver (in control plane)
        ↓
etcd lookup
        ↓
Response: Pod list
        ↓
User sees pods
```

### Pod → Pod Communication

```
Pod A in Node 1 (10.0.1.101)
        ↓
Wants to connect to Service: my-service
        ↓
DNS query (coredns pod)
        ↓
CoreDNS returns: my-service = 10.100.0.5 (ClusterIP)
        ↓
kubelet checks iptables rules (kube-proxy)
        ↓
Routes to one of 3 backend pods (load balanced)
        ↓
Pod B, Pod C, or Pod D (pod selector)
        ↓
Connection established
```

### Pod → AWS Service (e.g., S3)

```
Pod A (10.0.1.101)
        ↓
Wants to reach S3 bucket
        ↓
Pod uses VPC endpoint (no internet needed)
        ↓
VPC endpoint routes to S3
        ↓
Or: Pod routes through NAT Gateway to internet
        ↓
IAM role (via IRSA) provides credentials
        ↓
S3 request authenticated
        ↓
S3 response
```

---

## EKS API Server Endpoint Access

### Public Endpoint
```yaml
# Everyone can access (but requires IAM auth)
# Default: kubectl reaches from anywhere
# Requires correct IAM credentials in kubeconfig

kubectl get nodes  # Works from local machine
```

### Private Endpoint
```yaml
# Only accessible from VPC
# Requires VPC Endpoint or EC2 instance in VPC
# More secure: no internet exposure

# Setup:
# 1. Edit cluster endpoint access
# 2. Enable private endpoint
# 3. Add your IP to security group
```

### Both Endpoints (Recommended)
```yaml
# Public endpoint for dev convenience
# Private endpoint for production security
# Both secured by IAM
```

---

## Your Interview Setup

### What You're Managing

```
Environments: 4 (DEV, QA, Staging, PROD)
├── DEV Cluster
│   ├── 1-2 t3.medium nodes
│   ├── 3-4 namespaces
│   ├── 50-100 pods
│   └── Quick deployment cycles
├── QA Cluster
│   ├── 2-3 t3.large nodes
│   ├── 5-8 namespaces
│   ├── 100-150 pods
│   └── Load testing capability
├── Staging Cluster
│   ├── 2-3 t3.large nodes
│   ├── 5-8 namespaces
│   ├── Same config as production
│   └── Pre-production testing
└── Production Cluster (HA)
    ├── 6-12 c5.xlarge nodes
    ├── 8-10 namespaces
    ├── 300-500 pods
    ├── Spread across 3 AZs
    ├── Auto-scaling enabled
    └── Monitoring and alerting
```

### Key Configuration Points

**For each cluster, know:**
- VPC CIDR and subnet sizing
- Security group rules
- Auto-scaling thresholds
- Add-on versions
- IAM roles for node groups
- Logging and monitoring setup
- Backup and disaster recovery

---

## Common Interview Questions on EKS Basics

### Q1: "Explain your EKS architecture"

**Answer:**
"I manage a multi-cluster EKS architecture across 4 environments: DEV, QA, Staging, and Production. Each cluster has its own VPC and Kubernetes control plane managed by AWS. The data plane consists of EC2 nodes organized in node groups with auto-scaling. DEV is lightweight (1-2 t3.medium nodes), while Production runs 6-12 c5.xlarge nodes across 3 AZs for high availability. All clusters use the same VPC CNI add-on for pod networking, giving each pod a real VPC IP. This enables direct communication with AWS services without NAT overhead."

### Q2: "What happens when the control plane has issues?"

**Answer:**
"AWS manages the control plane with 99.95% SLA and automatic replication across multiple AZs. If there's a control plane issue, AWS automatically recovers it. For the user, this means: existing pods continue running (kubelet keeps them alive), but you can't manage them (kubectl commands fail). Once control plane recovers, kubectl works again, but workloads are unaffected. This is why we don't worry much about control plane failures - AWS handles it."

### Q3: "Why multiple AZs?"

**Answer:**
"Multiple AZs provide fault tolerance. If one AZ fails (power outage, network issue), the other AZs still have nodes and pods. Kubernetes schedules replicas across AZs automatically (via pod anti-affinity). So if AZ-1 fails, pods are still running on AZ-2 and AZ-3. For production, we require 3 replicas minimum and spread them across 3 AZs. This ensures we can survive 1-2 AZ failures without service degradation."

### Q4: "How does pod networking work?"

**Answer:**
"EKS uses the AWS VPC CNI plugin, which assigns real VPC IP addresses to pods, not overlay IPs. Each node gets multiple ENIs, and pods run in the node's VPC subnets. This means pods can directly communicate with AWS services like RDS, ElastiCache, without NAT. It also simplifies security groups - we just use VPC security groups. The trade-off is that a node is limited by the number of ENIs it can attach (e.g., c5.large can attach 4 ENIs, each with multiple IPs)."

### Q5: "How do you scale your clusters?"

**Answer:**
"We use Cluster Autoscaler which monitors pending pods. When a pod can't be scheduled (no node has resources), the autoscaler triggers the ASG to launch new nodes. This usually takes 1-3 minutes. For faster scaling of specific workloads, we can use Karpenter, which is more efficient. For DEV, manual scaling is fine. For production, autoscaling is essential because we can't predict traffic spikes. We also use pod-level autoscaling with HPA (Horizontal Pod Autoscaler) which scales replicas based on CPU/memory."

---

## Quick Facts to Remember

- EKS control plane is managed by AWS (99.95% SLA)
- You manage the data plane (nodes and pods)
- Multi-AZ deployment is critical for HA
- VPC CNI gives pods real VPC IPs
- Auto-scaling requires Cluster Autoscaler or Karpenter
- Security groups control node-to-node and pod-to-AWS traffic
- etcd is AWS-managed and backed up automatically
- API endpoint requires IAM authentication (even public endpoint)

