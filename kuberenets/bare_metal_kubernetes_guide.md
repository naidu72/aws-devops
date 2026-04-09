# Bare Metal Kubernetes - Self-Managed Clusters

## Module Overview

**Learning Objectives:**
- Understand cluster bootstrap with kubeadm
- Master CNI choice (Calico vs Cilium) for on-prem
- Design high-availability control plane
- Plan storage for self-managed clusters
- Handle networking without cloud provider
- Troubleshoot on-prem specific issues

**Estimated Study Time:** 8-10 hours
**Context:** On-prem deployments less common in startups, but critical for enterprises

---

## Part 1: Cluster Bootstrap with kubeadm

### 1.1 Prerequisites

**Minimum Hardware per Node:**
- CPU: 2+ cores
- RAM: 2GB (control plane), 1GB (worker)
- Disk: 20GB+
- Network: Shared subnet, all nodes reachable

**Required Software:**
- OS: Linux (Ubuntu 20.04 LTS recommended)
- Container Runtime: docker or containerd
- kubeadm, kubelet, kubectl

**Network Planning:**
- Cluster CIDR: 10.0.0.0/8 (pods)
- Service CIDR: 10.96.0.0/12 (services)
- Node IPs: 192.168.1.0/24 (static, no DHCP for control planes)

### 1.2 Install Prerequisites

```bash
# On all nodes (master & worker)

# Install container runtime (containerd)
apt-get update
apt-get install -y containerd.io

# Configure containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
systemctl restart containerd

# Install kubeadm, kubelet, kubectl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg \
  https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
  https://apt.kubernetes.io/ kubernetes-xenial main" | \
  tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubeadm=1.28.0-00 kubelet=1.28.0-00 kubectl=1.28.0-00

# Enable kubelet service
systemctl enable --now kubelet
```

### 1.3 Initialize Control Plane

```bash
# On first master node:
kubeadm init \
  --apiserver-advertise-address=192.168.1.100 \
  --pod-network-cidr=10.0.0.0/8 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.28.0

# Output includes join command for worker nodes (save this!)

# Setup kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 1.4 Install CNI Plugin

**Choose Calico or Cilium (discussed in Part 2)**

```bash
# Example: Calico
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml | kubectl apply -f -

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=300s
```

### 1.5 Join Worker Nodes

```bash
# On worker nodes: use the join command from init output
kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxx
```

---

## Part 2: CNI Plugin Choice (Critical for On-Prem)

### 2.1 Calico (Most Popular On-Prem)

**Architecture:**
- Uses BGP for routing (dynamic, scalable)
- iptables/eBPF for network policies
- No overlay network (direct routing)

**Deployment:**
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

**Configuration:**
```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.0.0.0/8
      encapsulation: VXLANMode  # or BGP for routing
      natOutgoing: Enabled
```

**Advantages:**
- BGP routing (no encapsulation overhead)
- Strong network policies (L3-L4)
- Proven at scale (used by big enterprises)
- Supports on-prem, cloud, hybrid

**Disadvantages:**
- Requires BGP knowledge
- Initial setup complexity
- Limited L7 policies

**Interview Q:** "You have 500 on-prem servers in 3 datacenters. Design the cluster networking."
**Answer:**
- Calico with BGP
- Configure BGP peers per datacenter
- Pod CIDR per datacenter (10.1.0.0/16, 10.2.0.0/16, 10.3.0.0/16)
- Network policies for security (default deny, allow by label)

### 2.2 Cilium (High Performance Alternative)

**Architecture:**
- eBPF-based (kernel-level, fast)
- Can replace kube-proxy
- Advanced L7 policies, encryption

**Deployment:**
```bash
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=strict \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

**Advantages:**
- High performance (kernel-level processing)
- L7 policies (inspect HTTP, gRPC)
- Built-in Hubble observability
- Modern approach

**Disadvantages:**
- Requires kernel 5.8+ (eBPF)
- Younger project (less battle-tested than Calico)
- Learning curve (eBPF concepts)

**Comparison:**
| Feature | Calico | Cilium |
|---------|--------|--------|
| **Maturity** | Very proven | Growing popularity |
| **Performance** | Good (iptables) | Excellent (eBPF) |
| **L7 Policies** | Limited | Full support |
| **Kernel Requirement** | 2.6+ | 5.8+ |
| **Setup Complexity** | Moderate (BGP) | Low (Helm) |

**Use Calico if:** Traditional setup, BGP available, proven stability needed
**Use Cilium if:** Modern cluster, high performance needed, L7 policies required

---

## Part 3: High-Availability Control Plane

### 3.1 Single Master (Dev/Test Only)

```
┌──────────────────┐
│  Master Node     │
│  - API Server    │
│  - etcd          │
│  - Controller    │
│  - Scheduler     │
└──────────────────┘
        ↓
   ┌────────────┐
   │ 3 Workers  │
   └────────────┘
```

**Risk:** Single point of failure (master down = cluster down)

### 3.2 HA Control Plane (3+ Masters)

```
┌──────────────────┐
│  Master Node 1   │
│  - API Server    │
│  - etcd member 1 │
│  - Controller    │
│  - Scheduler     │
└──────────────────┘
        ↓
┌──────────────────┐      ┌──────────────────┐
│  Master Node 2   │      │  Master Node 3   │
│  - API Server    │      │  - API Server    │
│  - etcd member 2 │──────│  - etcd member 3 │
│  - Controller    │      │  - Controller    │
│  - Scheduler     │      │  - Scheduler     │
└──────────────────┘      └──────────────────┘
        ↑                         ↑
        └─────┬──────────────────┘
              ↓
        ┌──────────────┐
        │ Load Balancer│  (nginx, HAProxy, F5)
        └──────────────┘
              ↑
        ┌──────────────────┐
        │  API Endpoint    │
        │  (clients connect)
        └──────────────────┘
```

**Setup Steps:**

1. **Initialize first master:**
   ```bash
   kubeadm init --control-plane-endpoint=apiserver.example.com:6443 \
     --pod-network-cidr=10.0.0.0/8
   ```

2. **Generate certs for additional masters:**
   ```bash
   kubeadm certs --config kubeadm-config.yaml

   # Copy to other masters
   scp -r /etc/kubernetes/pki master2:/etc/kubernetes/
   ```

3. **Initialize subsequent masters:**
   ```bash
   kubeadm join 192.168.1.100:6443 \
     --token abcdef.0123456789abcdef \
     --discovery-token-ca-cert-hash sha256:xxxxx \
     --control-plane \
     --certificate-key xxxxxxx
   ```

4. **Setup Load Balancer:**
   ```nginx
   upstream apiserver {
       server master1.example.com:6443 weight=1 max_fails=3 fail_timeout=10s;
       server master2.example.com:6443 weight=1 max_fails=3 fail_timeout=10s;
       server master3.example.com:6443 weight=1 max_fails=3 fail_timeout=10s;
   }

   server {
       listen 6443 ssl;
       server_name apiserver.example.com;
       
       ssl_certificate /etc/pki/tls/certs/apiserver.crt;
       ssl_certificate_key /etc/pki/tls/private/apiserver.key;
       
       location / {
           proxy_pass https://apiserver;
       }
   }
   ```

**etcd Redundancy:**
```bash
# etcd is distributed consensus database
# 3+ members for HA
# If 1 member down: cluster survives
# If 2+ members down: cluster fails

# Check etcd health
kubectl exec -n kube-system etcd-master1 -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

---

## Part 4: Storage for On-Prem

### 4.1 Local Storage (Direct Node Disk)

**Simplest but limited:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  local:
    path: /mnt/data  # Direct node disk
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker1
```

**Limitations:**
- Pod tied to specific node
- Data lost if node fails
- No replication

### 4.2 NFS Storage

**Shared storage via NFS:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteMany  # Multiple pods can write simultaneously
  nfs:
    server: nfs-server.example.com
    path: "/exports/data"
```

**Setup NFS Server:**
```bash
# On NFS server
apt-get install nfs-kernel-server

# Configure export
echo "/exports/data *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
exportfs -ra
```

**Advantages:**
- Shared storage (multiple pods)
- Simple to setup
- Cross-node access

**Disadvantages:**
- Single point of failure (NFS server)
- Performance bottleneck
- No replication

### 4.3 Ceph (Distributed Storage)

**Enterprise-grade distributed storage:**

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: ceph/ceph:v17.2.0
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
  osd:
    count: 3
```

**Advantages:**
- HA (data replicated across nodes)
- Self-healing (detects corrupted data)
- Scalable (add nodes → add storage)
- Performance (parallel access)

**Disadvantages:**
- Complex setup & management
- Resource overhead (replication, healing)
- Learning curve

**Best Practice:** Ceph for production on-prem clusters

---

## Part 5: Networking Challenges On-Prem

### 5.1 IP Address Management

**Challenge:** How many IPs per node?

**Planning:**
```
Total VMs needed:
- 3 masters × 1 core each = 3 VMs
- 10 workers × 50 pods max = pod capacity
- Pod CIDR: 10.0.0.0/16 = 65,536 IPs
- Service CIDR: 10.96.0.0/12 = 1,048,576 IPs

Sufficient for medium clusters (100+ nodes)
```

### 5.2 External Load Balancer

**Challenge:** No cloud LB (like AWS ALB)

**Solution 1: MetalLB (Kubernetes Load Balancer)**
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb.yaml

# Configure pool of IPs
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 203.0.113.0/24  # Public IPs for LB
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - example
EOF
```

**Usage:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: myapp
```

**Result:** MetalLB assigns external IP from pool

**Solution 2: HAProxy/Nginx Ingress**
```bash
helm install nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### 5.3 Ingress without Cloud

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx  # or haproxy
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app
            port:
              number: 8080
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
```

**DNS:**
```
app.example.com → ingress-nginx service IP (public)
                → routes to backend pods
```

---

## Part 6: On-Prem Cluster Checklist

### Planning Phase:
- [ ] Network topology (all nodes reachable)
- [ ] IP addressing (cluster CIDR, pod CIDR, services)
- [ ] Hardware (CPU, RAM, disk per node)
- [ ] Storage strategy (local, NFS, Ceph)
- [ ] High availability (1 vs 3+ masters)
- [ ] Load balancer (MetalLB or external)

### Installation Phase:
- [ ] Install container runtime (containerd)
- [ ] Install kubeadm, kubelet, kubectl
- [ ] Initialize masters with kubeadm
- [ ] Join worker nodes
- [ ] Install CNI (Calico recommended)

### Post-Installation:
- [ ] Install storage provisioner (Ceph, NFS)
- [ ] Deploy load balancer (MetalLB)
- [ ] Deploy ingress controller (nginx)
- [ ] Setup DNS
- [ ] Enable monitoring (Prometheus)
- [ ] Setup logging (ELK, Loki)

### HA Setup (if 3+ masters):
- [ ] Setup load balancer (HAProxy, nginx, F5)
- [ ] Configure etcd cluster
- [ ] Test failure scenarios

---

## Part 7: Comparison: On-Prem vs Cloud

| Aspect | On-Prem | AWS EKS | Azure AKS |
|--------|---------|---------|-----------|
| **Setup Time** | Days (manual) | Hours | Hours |
| **Operational Burden** | High (manage everything) | Low (AWS manages control plane) | Low |
| **Upgrades** | Manual (downtime possible) | Managed (no downtime) | Managed |
| **High Availability** | Manual (HA setup required) | Built-in (multi-AZ) | Built-in |
| **Storage** | Manual (NFS, Ceph) | Managed (EBS, EFS) | Managed (Disk, Files) |
| **Network** | Complex (manual config) | Simplified (VPC integration) | Simplified |
| **Cost** | Infrastructure + Management | Infrastructure + Control Plane | Infrastructure + Control Plane |
| **Suitable For** | Large enterprises, on-prem compliance | Startups, AWS-centric | Enterprises, Azure-centric |

---

## Key Takeaways

1. **Bootstrap:** kubeadm handles cluster init, joining nodes
2. **CNI Choice:** Calico (proven) or Cilium (modern)
3. **HA Control Plane:** 3+ masters, etcd cluster, load balancer
4. **Storage:** Local (simple), NFS (shared), Ceph (enterprise)
5. **Networking:** Plan IP CIDR, setup external LB, configure DNS
6. **Complexity:** On-prem requires more operational knowledge than cloud

---

## Study Plan (Next 2 Hours)

1. **Theory (45 min):** kubeadm, CNI choice, HA control plane
2. **Design (30 min):** Design on-prem cluster for 500-node enterprise
3. **Troubleshooting (30 min):** Common on-prem issues
4. **Comparison (15 min):** On-prem vs cloud trade-offs
