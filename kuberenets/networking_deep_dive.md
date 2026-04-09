# PART 3: KUBERNETES NETWORKING DEEP DIVE

## CNI PLUGINS: COMPREHENSIVE COMPARISON

### 1. AMAZON VPC CNI (workspace focus)

**Architecture:**
```
AWS VPC Network (10.0.0.0/16)
    ↓
Each node gets multiple ENIs (Elastic Network Interfaces)
    ↓
Each ENI gets multiple secondary IPs
    ↓
AWS VPC CNI assigns those secondary IPs to pods
    ↓
Result: Pods get actual VPC IPs (10.0.x.x), not overlay IPs
```

**Advantages:**
- Native AWS integration (no overlay overhead)
- Lower latency (direct VPC routing)
- AWS security groups per pod
- DNS propagates with real IPs

**Disadvantages:**
- Limited IPs per node (determined by instance type)
- IP exhaustion at scale (common problem)
- WARM_IP_TARGET tuning complexity
- Tied to AWS (not portable)

**Key Configuration:**
```bash
# WARM_IP_TARGET: Pre-allocated IPs per node
env:
- name: WARM_IP_TARGET
  value: "5"

# WARM_PREFIX_TARGET: Pre-allocated IP prefixes
env:
- name: WARM_PREFIX_TARGET
  value: "1"  # Newer, more efficient

# Maximum IPs per node depends on instance type:
# t3.small: 11 IPs per ENI × 2 ENIs = 22 IPs
# t3.medium: 10 IPs per ENI × 3 ENIs = 30 IPs
# m5.large: 10 IPs per ENI × 3 ENIs = 30 IPs

# Problem: WARM_IP_TARGET=5 on large cluster
# 100 nodes × 5 warm IPs = 500 IPs reserved
# Subnet /24 = 250 usable IPs
# Result: IP exhaustion!

# Solution: Prefix Delegation mode
env:
- name: ENABLE_PREFIX_DELEGATION
  value: "true"
```

**Real Scenario Interview:**
"You're seeing pod scheduling failures on your 100-node EKS cluster. 
Nodes have < 1 pod each, but new pods won't schedule. What's happening?"

Answer: "IP exhaustion from WARM_IP_TARGET. Each node reserves warm IPs even if empty.
100 nodes × 5 warm IPs = 500 IPs reserved, but only 250 available in /24 subnet.
Solutions: Enable prefix delegation, increase VPC CIDR, or tune WARM_IP_TARGET."

---

### 2. CALICO - Network Policies + BGP

**Architecture:**
```
Calico agent on every node (DaemonSet)
    ↓
Each node announces CIDR block (via BGP)
    ↓
Routers learn routes dynamically (BGP)
    ↓
Cross-node traffic routed directly (no tunnel)
    ↓
Network policies enforced via iptables rules on each node
```

**Features:**
- **Network Policies:** Full ingress/egress enforcement
- **BGP Routing:** Dynamic, no overlay overhead
- **iptables vs eBPF:** Old = iptables, new = eBPF mode
- **Cross-region:** BGP scales across regions

**Advantages:**
- Mature, well-tested
- Excellent network policy support
- No overlay (lower latency)
- Works on any infrastructure (not AWS-specific)

**Disadvantages:**
- Requires BGP setup (operational overhead)
- Complex debugging (iptables rules hard to trace)
- Higher CPU usage than Cilium (for policy enforcement)

**Installation:**
```bash
# Install Calico
helm repo add calico https://docs.tigera.io/calico/charts
helm install calico calico/tigera-operator --namespace tigera-system --create-namespace

# Enable eBPF mode (newer, more efficient)
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF"}}}'
```

**Interview Point:**
"Design network policies for a multi-tier microservices architecture (frontend, API, database)."

Answer: "Calico allows fine-grained policies. Default deny, then allow specific routes:
- Ingress: Allow external → frontend (port 80)
- Ingress: Allow frontend → api (port 8080)
- Ingress: Allow api → database (port 5432)
- Egress: Allow egress to external (port 443 for HTTPS)"

---

### 3. CILIUM - eBPF + Advanced Features

**Architecture:**
```
Cilium agent on every node (DaemonSet)
    ↓
eBPF programs loaded into kernel
    ↓
Kernel-level interception (no iptables overhead)
    ↓
Hugely efficient + powerful
    ↓
Hubble for observability (network flow visualization)
```

**Features:**
- **eBPF:** Kernel-level packet processing (super fast)
- **L7 Policies:** HTTP-aware policies (allow GET /api/users only)
- **Hubble:** Built-in network observability
- **Service Mesh:** Can replace Istio for some use cases

**Advantages:**
- Extremely fast (eBPF vs iptables)
- Advanced L7 policies
- Built-in observability (Hubble)
- More efficient than Calico iptables mode

**Disadvantages:**
- Newer, less mature than Calico
- Requires kernel 5.8+ (eBPF support)
- Learning curve (eBPF concepts)
- Fewer production deployments (fewer battle scars)

**Installation:**
```bash
# Install Cilium
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set cni.mode=direct \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Access Hubble UI for network flows visualization
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
# Visit: http://localhost:12000
```

**Hubble Example Query:**
```bash
# See all flows between frontend and api
hubble observe --pod frontend --pod api

# Output: Shows source IP → dest IP, port, protocol, status
```

**Interview Advantage:**
"Cilium provides automatic network visualization via Hubble. Can see all pod-to-pod 
communication flows without manual logging. Useful for debugging connectivity issues."

---

### CNI COMPARISON TABLE

| Feature | Amazon VPC CNI | Calico | Cilium |
|---------|----------------|--------|---------|
| **Type** | AWS-native | CNI + Policies | CNI + eBPF |
| **Routing** | VPC native | BGP | Direct |
| **Network Policies** | Via SGs | Full support | Full + L7 |
| **Performance** | Very fast | Good | Excellent |
| **Complexity** | Low (AWS-integrated) | Medium (BGP setup) | Medium (eBPF) |
| **Observability** | CloudWatch | Manual | Hubble (built-in) |
| **Cost** | Free (AWS infra) | Free | Free |
| **Multi-cloud** | AWS only | Yes | Yes |
| **Maturity** | Very mature | Very mature | Newer |
| **Best For** | EKS | Multi-cloud, policies | Performance, L7 policies |

---

## NETWORK POLICY DEEP DIVE

### Anatomy of a NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  # Which pods does this policy apply to?
  podSelector:
    matchLabels:
      app: api-server

  # Policy types: what to control?
  policyTypes:
  - Ingress  # Control incoming traffic
  - Egress   # Control outgoing traffic

  # Ingress rules (who can talk TO matching pods?)
  ingress:
  - from:
    # Option 1: Pods in same namespace
    - podSelector:
        matchLabels:
          app: frontend
    
    # Option 2: Pods in specific namespace
    - podSelector:
        matchLabels:
          app: worker
      namespaceSelector:
        matchLabels:
          name: background-jobs
    
    # Option 3: IP CIDR blocks
    - ipBlock:
        cidr: 192.168.1.0/24
        except:
        - 192.168.1.1/32
    
    # Define allowed ports
    ports:
    - protocol: TCP
      port: 8080
    - protocol: TCP
      port: 8443
  
  # Egress rules (where can matching pods talk TO?)
  egress:
  - to:
    # Allow egress to database pods
    - podSelector:
        matchLabels:
          app: postgres
    
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow egress to DNS (often required!)
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

### Common NetworkPolicy Patterns

#### Pattern 1: Default Deny All + Allow Specific

```yaml
# Step 1: Deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # Matches ALL pods
  policyTypes:
  - Ingress
  ingress: []  # Empty = deny all

---
# Step 2: Allow specific (frontend)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Interview Value:** "This is zero-trust networking. Default deny prevents 
accidental exposure, then explicitly allow required connections."

---

#### Pattern 2: Cross-Namespace Communication

```yaml
# In production namespace, allow ingress from default namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-default
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  
  policyTypes:
  - Ingress
  
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: default  # Namespace label
    ports:
    - protocol: TCP
      port: 8080

---
# REQUIREMENT: Label the namespace
kubectl label namespace default name=default
```

---

#### Pattern 3: Egress Controls (Prevent Data Exfiltration)

```yaml
# Strict egress policy: only allow to database, DNS, and approved external
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: strict-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: worker
  
  policyTypes:
  - Egress
  
  # Allow TO database
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  
  # Allow TO DNS (required!)
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  
  # Allow TO external API (specific IP)
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
    ports:
    - protocol: TCP
      port: 443
```

**Security Value:** "Prevents compromised pods from exfiltrating data. Even if 
attacker gets in, they can't call home (egress blocked)."

---

### NetworkPolicy Debugging

```bash
# Check if policies applied
kubectl get networkpolicy -n production

# Describe policy
kubectl describe networkpolicy my-policy -n production

# Check which pods match selector
kubectl get pods -n production -L app
# Verify labels match podSelector

# Test connectivity from one pod
kubectl exec -it <source-pod> -n default -- bash
pod$ nc -zv payment-service.production 8080
# If "Connection refused" → policy blocking

# Check node's iptables rules (if using iptables mode)
kubectl debug node/my-node -it --image=ubuntu
node# iptables -L -n | grep NAMESPACE

# For Cilium, use Hubble
kubectl port-forward -n kube-system svc/hubble-relay 4245:4245
hubble observe --pod default/source-pod --pod production/target-pod

# Common issues:
# 1. Missing DNS policy (port 53 UDP)
# 2. Source pod not matching any "from" selector
# 3. Port number mismatch (service port vs container port)
# 4. Namespace label not set
```

---

## SERVICE DISCOVERY INTERNALS

### Complete Flow: Pod A → Pod B Communication

```
Pod A (wants to reach service-b.production.svc.cluster.local:8080)
    ↓
1. Pod does DNS query (systemd-resolved or musl libc)
    ↓
2. Query hits CoreDNS (port 53, UDP) via cluster DNS IP (10.0.0.10)
    ↓
3. CoreDNS watches Service objects, returns cluster IP
    Service service-b spec.clusterIP: 10.0.1.200
    Response: 10.0.1.200
    ↓
4. Pod connects to 10.0.1.200:8080
    ↓
5. kube-proxy running on node has watched Service + EndpointSlices
    EndpointSlices show: service-b has endpoints 10.0.2.1, 10.0.2.2, 10.0.2.3
    ↓
6. kube-proxy creates iptables rule:
    DNAT 10.0.1.200:8080 → (10.0.2.1:8080 OR 10.0.2.2:8080 OR 10.0.2.3:8080)
    ↓
7. kube-proxy load-balances (round-robin) across endpoints
    ↓
8. Packet sent to actual pod IP (10.0.2.1, etc.)
    ↓
9. Kernel routes directly to target pod (via CNI)
    ↓
10. Pod B receives connection on port 8080
    ↓
SUCCESS
```

---

### kube-proxy Modes Comparison

| Mode | Mechanism | Performance | Scalability | Debugging |
|------|-----------|------------|------------|-----------|
| **iptables** | Kernel NAT via iptables | High | OK | Difficult (many rules) |
| **IPVS** | Kernel module (IP Virtual Server) | Very High | Excellent | Medium (cleaner rules) |
| **userspace** | Userland proxy | Low | Poor | Easy (simple) |

**Default:** iptables (stable, well-tested)
**Recommended:** IPVS (large clusters > 1000 services)

---

### EndpointSlices (Modern, Replaces Endpoints)

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: service-b-abc123
  namespace: production
  labels:
    kubernetes.io/service-name: service-b

addressType: IPv4

endpoints:
- addresses:
  - "10.0.2.1"
  conditions:
    ready: true
  targetRef:
    kind: Pod
    name: service-b-pod-1

- addresses:
  - "10.0.2.2"
  conditions:
    ready: true
  targetRef:
    kind: Pod
    name: service-b-pod-2

ports:
- name: http
  protocol: TCP
  port: 8080

# Why EndpointSlices?
# - More scalable (split into slices if too many endpoints)
# - Atomic updates (each slice updates independently)
# - Faster updates (only changed slice updates)
```

---

## INTERVIEW SCENARIOS

### Scenario 1: "Pods on different nodes can't communicate. Debug it."

**Step-by-step debugging:**

```bash
# 1. Verify pods are running
kubectl get pods -o wide
# Check Pod IPs and which nodes they're on

# 2. Test direct IP connectivity (bypasses service DNS)
kubectl exec -it pod-a -- ping pod-b-ip
# If fails → network/CNI issue
# If works → service discovery issue

# 3. Test DNS resolution
kubectl exec -it pod-a -- nslookup service-b.production.svc.cluster.local
# If fails → CoreDNS issue or network policy blocking DNS (UDP 53)

# 4. Check service endpoints
kubectl get endpoints service-b -n production
# If empty → pod labels don't match service selector

# 5. Check NetworkPolicies
kubectl get networkpolicy -n production
# Check if blocking traffic

# 6. Check kube-proxy (if using iptables)
sudo iptables -L -n | grep <service-ip>
# Should show DNAT rules for service

# 7. Check node routing
kubectl debug node/worker-1 -it --image=ubuntu
node# ip route | grep pod-b-subnet
# Should show route to pod's node/subnet
```

---

### Scenario 2: "Design network policies for a payment system"

**Requirements:**
- Frontend (public, internet-facing)
- Payment API (internal only)
- Payment processor (external, HTTPS only)
- Database (internal only)

**Policy Design:**

```yaml
---
# Namespace labels for policies
apiVersion: v1
kind: Namespace
metadata:
  name: payment
  labels:
    name: payment

---
# 1. Frontend ingress (allow from internet)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-ingress
  namespace: payment
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0  # ANY source (internet)
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443

---
# 2. Frontend egress (only to API)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: payment
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: payment-api
    ports:
    - protocol: TCP
      port: 8080
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# 3. Payment API ingress (only from frontend)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-ingress
  namespace: payment
spec:
  podSelector:
    matchLabels:
      app: payment-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# 4. Payment API egress (to DB + processor)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress
  namespace: payment
spec:
  podSelector:
    matchLabels:
      app: payment-api
  policyTypes:
  - Egress
  egress:
  # To database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  
  # To external processor (HTTPS only, specific IPs)
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24  # Payment processor IPs
    ports:
    - protocol: TCP
      port: 443
  
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# 5. Database ingress (only from API)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-ingress
  namespace: payment
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: payment-api
    ports:
    - protocol: TCP
      port: 5432

---
# 6. Database egress (allow backups to S3)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-egress
  namespace: payment
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Egress
  egress:
  # To S3 API endpoint
  - to:
    - ipBlock:
        cidr: 52.0.0.0/8  # AWS S3 IP ranges
    ports:
    - protocol: TCP
      port: 443
  
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**Interview Value:** "This demonstrates understanding of defense-in-depth 
(multiple layers), zero-trust (explicit allow), and least privilege."

---

### Scenario 3: "Your company uses Amazon VPC CNI. You're getting IP exhaustion. How do you fix it?"

**Root Cause Analysis:**
1. Check current WARM_IP_TARGET
2. Calculate pods per node
3. Calculate total IPs needed
4. Compare with subnet size

**Solution (in priority order):**

```bash
# 1. BEST: Enable Prefix Delegation
kubectl set env daemonset/aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1

# 2. If not enough, increase VPC CIDR (add secondary CIDR)
# (Do this in AWS console or Terraform)

# 3. Last resort: Reduce WARM_IP_TARGET
kubectl set env daemonset/aws-node -n kube-system \
  WARM_IP_TARGET=1

# 4. Verify fix
kubectl describe daemonset -n kube-system aws-node | grep -i warm
kubectl get nodes -o wide
# New pods should schedule now
```

---

## KEY INTERVIEW TAKEAWAYS

1. **CNI Choice Matters:** EKS uses Amazon VPC CNI (IP exhaustion risk), 
   but Calico/Cilium available for multi-cloud

2. **NetworkPolicies = Security:** Zero-trust networking prevents lateral movement

3. **Service Discovery:** DNS + kube-proxy + endpoints create seamless pod communication

4. **Debugging Skills:** Know how to test DNS, connectivity, policies, and kube-proxy rules

5. **Scale Awareness:** WARM_IP_TARGET, prefix delegation, EndpointSlices all designed for scale

---

