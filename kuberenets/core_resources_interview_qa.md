# CORE KUBERNETES RESOURCES - INTERVIEW Q&A

## Q1: "Explain the difference between a Deployment and a StatefulSet. When would you use each?"

### FULL ANSWER:

**Deployment:**
- **Purpose:** For stateless applications where all replicas are identical and interchangeable
- **Pod naming:** Random suffixes (webapp-a1b2c, webapp-d3e4f) - NOT guaranteed to be the same
- **DNS:** All pods behind single service VIP (cluster IP)
- **Storage:** Each pod gets own tmpfs/emptyDir (lost on pod restart)
- **Scaling:** All pods created/destroyed in parallel for speed
- **Updates:** Rolling updates with new pods created before old destroyed (zero downtime)
- **Use cases:** Web servers, APIs, microservices, stateless workloads
- **Data persistence:** Not suitable - no stable identity

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  # nginx-abc12, nginx-def45, nginx-ghi67 (random names each time)
```

---

**StatefulSet:**
- **Purpose:** For stateful applications requiring stable identity and persistent storage
- **Pod naming:** Ordinal indices (postgres-0, postgres-1, postgres-2) - GUARANTEED stable
- **DNS:** Each pod has stable DNS: `postgres-0.postgres.default.svc.cluster.local`
- **Storage:** Each pod bound to specific PVC (data persists across restarts)
- **Scaling:** Pods created one-at-a-time in order (0→1→2) for data consistency
- **Updates:** Sequential updates (pod 0, then pod 1, then pod 2) - safer for stateful
- **Use cases:** Databases (PostgreSQL, MySQL), caches (Redis), message queues (Kafka, RabbitMQ)
- **Data persistence:** Yes - PVCs bind to specific ordinals

**Example:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # Headless service (clusterIP: None)
  replicas: 3
  # postgres-0, postgres-1, postgres-2 (same names, predictable)
  # Each gets own PVC: postgres-storage-0, postgres-storage-1, postgres-storage-2
```

---

**Why This Matters (Interview Focus):**

| Aspect | Deployment | StatefulSet |
|--------|-----------|------------|
| **Pod Identity** | Random, ephemeral | Ordinal, stable |
| **DNS** | VIP (single endpoint) | Individual pod DNS |
| **Storage** | Temporary (emptyDir) | Persistent (PVC) |
| **Deployment Order** | Parallel | Sequential |
| **Data Loss Risk** | Expected (stateless) | Unacceptable (stateful) |
| **Replica Roles** | Identical | Often: 1 primary + N replicas |

---

### FOLLOW-UP QUESTIONS:

**Q: "What happens if I scale down a StatefulSet?"**
A: Pods deleted in reverse order (2→1→0). Importantly, PVCs are NOT deleted (data persists).
If you scale back up to 3, pods 0-1 reattach to existing PVCs with old data.

**Q: "Why do StatefulSets use a headless service?"**
A: Headless service (clusterIP: None) returns individual pod IPs instead of single VIP.
This allows apps to discover specific replicas (e.g., connect to postgres-0 for writes, postgres-1/-2 for reads).

**Q: "Can you use a Deployment for a database?"**
A: Technically yes, but VERY risky. Deployments don't guarantee pod identity, so scaling creates NEW pods.
If you attach PVCs, you get race conditions (multiple pods reading/writing same volume).
StatefulSets enforce sequential scaling to prevent this.

---

## Q2: "A pod is in CrashLoopBackOff state. Walk through your debugging steps."

### TROUBLESHOOTING WORKFLOW:

**Step 1: Examine pod status**
```bash
kubectl describe pod <pod-name>
# Look for:
# - Exit Code: 0 (clean exit) vs 1 (error)
# - Last State: Terminated (why did it exit?)
# - Events section: any clues?
```

**Step 2: Check recent logs**
```bash
kubectl logs <pod-name>
# If logs don't help, check previous attempt:
kubectl logs <pod-name> --previous
# (--previous shows logs from BEFORE the crash)
```

**Step 3: Check for init container failures**
```bash
kubectl describe pod <pod-name>
# Look for InitContainers section
# If init container failed, pod never reaches main container
# Check init container logs:
kubectl logs <pod-name> -c <init-container-name>
```

**Step 4: Verify resource constraints**
```bash
kubectl describe pod <pod-name>
# Look for:
# - Status: Pending? → Not enough resources on nodes
# - QoS Class: Guaranteed/Burstable/BestEffort
# - If OOMKilled: Pod exceeded memory limit
```

**Step 5: Check liveness/readiness probes**
```bash
kubectl describe pod <pod-name>
# Look for:
# - Liveness probe: httpGet /health failing?
# - Readiness probe: httpGet /ready failing?
# - If too aggressive (short initialDelaySeconds), pod crashes on startup
```

**Step 6: Verify mounts and permissions**
```bash
# If pod uses volumes, check mounts:
kubectl describe pod <pod-name> | grep -A 10 Mounts
# Common issues:
# - ConfigMap doesn't exist
# - Secret missing
# - Mount path permissions wrong
```

**Step 7: Check image availability**
```bash
# ImagePullBackOff?
kubectl describe pod <pod-name> | grep -i image
# - Image not found in registry?
# - Registry credentials missing?
# - Image tag wrong?
```

---

### COMMON CAUSES & FIXES:

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Exit code 0, keeps restarting | liveness probe failing | Adjust probe timing or fix app |
| Exit code 1, resource errors | Memory/CPU limit hit | Increase limits or optimize app |
| ImagePullBackOff | Image not in registry | Push image, fix tag, check credentials |
| Pending for 5+ mins | No node capacity | Add nodes or reduce resource requests |
| Init container failed | Config/secret missing | `kubectl apply` ConfigMap/Secret |
| Starts but crashes immediately | App error on startup | Check logs `--previous` |

---

### INTERVIEW RESPONSE TEMPLATE:

"First, I'd run `kubectl describe pod` to see the events. Then check logs with `kubectl logs` and `kubectl logs --previous`.
I'd look for init container failures, liveness probe failures, or OOMKilled events.
If nothing obvious, I'd check resource requests/limits against node capacity.
Finally, verify ConfigMaps and Secrets exist and mount paths are correct."

---

## Q3: "Explain how Kubernetes service discovery works."

### ARCHITECTURE:

```
Pod (client) wants to access another pod:
    ↓
Pod does DNS query: "user-service.default.svc.cluster.local"
    ↓
CoreDNS (DNS server in cluster) intercepts query
    ↓
CoreDNS checks Service object:
  apiVersion: v1
  kind: Service
  metadata:
    name: user-service
  spec:
    clusterIP: 10.0.1.100
    selector:
      app: user-api
    ports:
    - port: 8080
    ↓
CoreDNS returns: 10.0.1.100 (cluster IP / VIP)
    ↓
Client pod connects to 10.0.1.100:8080
    ↓
kube-proxy watches Service object and Endpoint Slices
    ↓
kube-proxy creates iptables/ipvs rules:
  DNAT 10.0.1.100:8080 → (10.0.2.1:8080 OR 10.0.2.2:8080 OR 10.0.2.3:8080)
    ↓
Load balances across actual pod IPs
    ↓
Traffic reaches pod
```

---

### KEY COMPONENTS:

**1. Service Object**
- Abstract definition of pods
- Label selector: `app: user-api`
- Cluster IP: Virtual IP (never changes)
- Port mapping: Service port 8080 → container port 8080

**2. CoreDNS**
- DNS server in cluster
- Watches Service objects
- Returns cluster IP for `servicename.namespace.svc.cluster.local`
- Also handles external DNS queries (fallback to upstream)

**3. Endpoint Slices (formerly Endpoints)**
- Dynamic list of pod IPs matching Service selector
- Updated when pods created/destroyed
- Example: Service label selector matches 3 pods → 3 endpoints

**4. kube-proxy**
- Runs on every node
- Watches Service and EndpointSlice objects
- Maintains routing rules (iptables, ipvs, or userspace proxy)
- Performs load balancing across endpoints

---

### DNS RESOLUTION EXAMPLES:

**Within same namespace:**
```bash
pod$ curl user-service:8080
# Resolves to: user-service.default.svc.cluster.local
```

**Across namespaces:**
```bash
pod$ curl user-service.production.svc.cluster.local:8080
# Resolves to: user-service service in production namespace
```

**Full FQDN:**
```bash
pod$ curl user-service.default.svc.cluster.local:8080
```

**Environment variables (legacy, not recommended):**
```bash
# Pod also gets env vars:
USER_SERVICE_SERVICE_HOST=10.0.1.100
USER_SERVICE_SERVICE_PORT=8080
# But DNS is preferred (more flexible)
```

---

### kube-proxy MODES:

| Mode | Type | Performance | Details |
|------|------|-------------|---------|
| **iptables** | Linux kernel | High | iptables rules, direct kernel NAT |
| **IPVS** | Linux kernel | Very High | ipvs (IP Virtual Server), more efficient |
| **Userspace** | User-level | Low | Slower, less common |
| **Nftables** | Newer kernel | High | Next-gen netfilter |

---

### LOAD BALANCING ALGORITHM:

By default, kube-proxy uses **round-robin** across endpoints:
```
Request 1 → 10.0.2.1
Request 2 → 10.0.2.2
Request 3 → 10.0.2.3
Request 4 → 10.0.2.1 (wraps around)
```

For session persistence, use `sessionAffinity: ClientIP` in Service spec.

---

### INTERVIEW FOCUS:

"CoreDNS watches Service objects and responds to DNS queries by returning the cluster IP.
kube-proxy watches the Service and its Endpoints, maintaining iptables rules that perform
NAT from the cluster IP to actual pod IPs. This provides service discovery and load balancing."

---

## Q4: "I have two pods in different namespaces that need to communicate. The connection is refused. Walk through your debugging steps."

### SYSTEMATIC DEBUGGING WORKFLOW:

**Step 1: Verify target service exists**
```bash
kubectl get svc -n production
# Is user-service in production namespace? If not, create it.
```

**Step 2: Verify endpoints (pods behind service)**
```bash
kubectl get endpoints -n production
# Example output:
# NAME            ENDPOINTS           AGE
# user-service    10.0.2.1:8080       2d

# If ENDPOINTS is empty, no pods match service selector
kubectl get pods -n production -L app
# Check: Do pods have label `app: user-api` (matching service selector)?
```

**Step 3: Test pod-to-pod connectivity**
```bash
# From source pod, try direct ping:
kubectl exec -it <source-pod> -n default -- bash
pod$ ping 10.0.2.1  # Direct IP
pod$ curl 10.0.2.1:8080  # Direct connection

# If this works → DNS/service issue
# If this fails → Network/policy issue
```

**Step 4: Test DNS resolution**
```bash
kubectl exec -it <source-pod> -n default -- bash
pod$ nslookup user-service.production.svc.cluster.local
# Should return cluster IP (10.0.x.x)

# If fails → CoreDNS issue or network policy blocking DNS
```

**Step 5: Check NetworkPolicies (most common culprit)**
```bash
# Check target namespace for ingress policies:
kubectl get networkpolicies -n production
kubectl describe networkpolicy -n production

# Common issue: Default deny ingress
# Solution: Add NetworkPolicy allowing ingress from source namespace

# Check source namespace for egress policies:
kubectl get networkpolicies -n default
# Check if egress to production blocked
```

**Step 6: Check security groups (if EKS)**
```bash
# AWS EKS uses security groups per pod
# Check pod security group:
kubectl describe pod <pod-name> -n production
# Look for annotations: vpc.amazonaws.com/pod-security-groups

# Verify security group rules allow inbound on port 8080
aws ec2 describe-security-groups --group-ids sg-xxxxx
```

**Step 7: Check RBAC (if service account permissions required)**
```bash
# If using IRSA or service account-based auth:
kubectl auth can-i get secrets -n production --as=system:serviceaccount:default:app-sa
# Verify source pod's service account can access target resources
```

**Step 8: Verify service type**
```bash
kubectl get svc user-service -n production
# type: ClusterIP? (correct for cross-namespace in cluster)
# Is it a headless service (clusterIP: None)? (different DNS behavior)
```

---

### COMMON CAUSES & FIXES:

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Connection refused` | Target service/pod not running | Check `kubectl get pods`, restart pod |
| `No route to host` | NetworkPolicy blocking | Add NetworkPolicy for ingress |
| `Temporary failure in name resolution` | DNS not resolving FQDN | Check CoreDNS, check NetworkPolicy for DNS (UDP 53) |
| `Connection timeout` | Security group blocking | Add security group rule for port 8080 |
| `Endpoints empty` | Pod labels don't match selector | Check pod labels: `kubectl get pods --show-labels` |
| `Port mismatch` | Service port != pod port | Check `kubectl describe svc`, verify containerPort |

---

### EXAMPLE FIX: Adding NetworkPolicy

If diagnosis shows NetworkPolicy blocking traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-default
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: user-api
  
  # Allow ingress from default namespace
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: default  # Requires: kubectl label namespace default name=default
    ports:
    - protocol: TCP
      port: 8080
```

---

### INTERVIEW RESPONSE TEMPLATE:

"I'd first verify the target service and its endpoints exist.
Then test direct pod-to-pod connectivity using the pod IP.
If that fails, I'd check NetworkPolicies and security groups.
If that works but DNS fails, I'd check for DNS-blocking policies.
Finally, I'd verify port numbers match between service and pod."

---

## Q5: "Explain Amazon VPC CNI's WARM_IP_TARGET and why it matters."

### WHAT IS WARM_IP_TARGET?

**Context:** Amazon VPC CNI assigns actual VPC IPs to pods (not overlay IPs).

```
Traditional CNI: Each pod gets virtual IP (10.244.x.x from overlay)
Amazon VPC CNI: Each pod gets real VPC IP (10.0.x.x from VPC subnet)
  ↓
Advantage: Native AWS integration, no overlay overhead, lower latency
Disadvantage: Limited IPs per node (each node has limited ENIs, each ENI has limited IPs)
```

---

**WARM_IP_TARGET:** Number of IPs pre-allocated but unassigned on each node.

### HOW IT WORKS:

```
Node starts:
  ↓
CNI allocates WARM_IP_TARGET IPs from VPC (e.g., 5 IPs)
  ↓
Pod starts:
  ↓
CNI assigns warm IP to pod (pool shrinks to 4 warm IPs)
  ↓
If warm pool drops below threshold, CNI allocates more (back to 5)
  ↓
Pod terminates:
  ↓
IP freed and returned to warm pool
```

### WHY IT MATTERS:

**1. Pod Startup Speed:**
- Without warm IPs: New pod waits for CNI to allocate IP from VPC (slow, 1-5 seconds)
- With warm IPs: New pod immediately assigned from warm pool (fast, milliseconds)
- **Interview Impact:** Higher WARM_IP_TARGET = faster pod scaling (good for HPA)

**2. IP Exhaustion Risk:**
- 2x vCPU node might support max 10 IPs (depends on instance type)
- If WARM_IP_TARGET=5, each node "reserves" 5 IPs even if only running 2 pods
- On large clusters, WARM_IP_TARGET=5 on 100 nodes = 500 IPs reserved (might exceed subnet CIDR)
- **Interview Impact:** Must balance between startup speed and IP availability

**3. Trade-offs:**

| WARM_IP_TARGET | Advantage | Disadvantage |
|-----------------|-----------|------------|
| High (5-10) | Fast pod startup | High IP reservation, easier to exhaust subnet |
| Low (1-2) | Efficient IP usage | Slow pod startup, HPA slower to scale |

---

### CONFIGURATION:

```bash
# Check current WARM_IP_TARGET:
kubectl describe daemonset -n kube-system aws-node
# Look for: WARM_IP_TARGET env var

# Modify (in values.yaml or directly):
env:
- name: WARM_IP_TARGET
  value: "5"  # Default

# Recalculate formula:
# Target pods per node = (Min IPs per ENI * Number of ENIs) - WARM_IP_TARGET
# E.g., t3.medium: 3 ENIs × 10 IPs/ENI - 5 warm = 25 pods max
```

---

### REAL SCENARIO:

**Company A (Startup):**
- 10 nodes, t3.small (10 IPs each)
- WARM_IP_TARGET = 5
- Pod startup critical for customer demos
- Total IPs reserved: 10 × 5 = 50 IPs (fine, plenty of space)
- ✓ Good choice: Fast scaling for HPA, plenty of IP space

**Company B (Large Enterprise):**
- 100 nodes, t3.small (10 IPs each)
- WARM_IP_TARGET = 5
- All nodes constantly full
- Total IPs reserved: 100 × 5 = 500 IPs
- Subnet CIDR /24 = 250 usable IPs
- ✗ Disaster: Can't schedule new pods (no warm IPs, IP exhaustion)
- Solution: Increase VPC CIDR, or switch to Prefix Delegation mode, or lower WARM_IP_TARGET

---

### SOLUTIONS TO IP EXHAUSTION:

**1. Prefix Delegation Mode:**
```bash
# Instead of allocating single IPs, allocate entire prefixes (/28)
# Reduces API calls, more efficient
env:
- name: ENABLE_PREFIX_DELEGATION
  value: "true"
- name: WARM_IP_TARGET
  value: "1"  # Now means 1 prefix, not 1 IP
```

**2. Increase VPC CIDR:**
```bash
# Secondary CIDR: Add 10.1.0.0/16 to VPC
# More IPs available for pod allocation
```

**3. Custom Networking:**
```bash
# Use custom ENI from different subnet
# Each node gets additional ENI with separate IP pool
```

---

### INTERVIEW FOCUS:

"Amazon VPC CNI pre-allocates warm IPs for fast pod startup.
Higher WARM_IP_TARGET means faster pod startup but higher IP reservation risk.
On large clusters, this can cause IP exhaustion if not tuned carefully.
Solutions include prefix delegation, increasing CIDR, or custom networking."

---

