# Multi-Architecture Deployments - Study Guide

## Quick Summary (1 hour)

### Architecture Types
- **x64 (amd64):** Intel/AMD 64-bit, most common
- **ARM64 (aarch64):** 64-bit ARM, AWS Graviton, Apple Silicon
- **x86 (i386):** 32-bit, legacy (rare in Kubernetes)

### Multi-Arch Image Strategy

**Step 1: Build for each architecture**
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

**Step 2: Push to registry (creates manifest list)**
```bash
# Result: Single tag points to architecture-specific images
myapp:latest
├── amd64: sha256:aaaaa...
├── arm64: sha256:bbbbb...
```

**Step 3: Kubernetes scheduler automatically selects correct image**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      nodeSelector:
        kubernetes.io/arch: amd64  # or arm64
      containers:
      - name: app
        image: myapp:latest  # Automatically uses correct architecture
```

### Common Interview Questions

**Q1: "Design multi-arch deployment for hybrid infrastructure (x64 + ARM64)"**
- Terraform: Create mixed node groups (x64 + ARM64)
- Dockerfile: Use buildx for multi-arch
- K8s: Node affinity or let scheduler choose
- Testing: Test on both architectures

**Q2: "How do you handle third-party dependencies not available for ARM?"**
- Use sidecar container (different architecture)
- Use emulation (slower but works)
- Switch to ARM-compatible library
- Build from source for ARM

**Q3: "Performance comparison x64 vs ARM64?"**
- ARM64 often: Lower power, good price/performance
- x64: More third-party library support
- Benchmark your specific app

### Practical: Multi-arch Deployment
```bash
# 1. Enable buildx
docker buildx create --use

# 2. Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag myregistry/myapp:v1.0 \
  --push \
  .

# 3. Verify manifest
docker manifest inspect myregistry/myapp:v1.0

# 4. Deploy to mixed cluster
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-arch-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myregistry/myapp:v1.0
EOF
```

---

# Scaling & High Availability - Study Guide

## Quick Summary (1 hour)

### HPA (Horizontal Pod Autoscaling)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up if > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 50  # Max scale down 50% at a time
```

### Cluster Autoscaler vs Karpenter

**Cluster Autoscaler:**
- Watches pending pods
- Scales up node group if needed
- Scales down idle nodes (after 10 min)
- Free/open-source

**Karpenter:**
- Faster scaling (seconds vs minutes)
- Bin-packing (consolidates pods)
- Native Spot instance support
- No need for ASG/node groups (more flexible)

### Pod Disruption Budgets (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2  # At least 2 pods must be running
  selector:
    matchLabels:
      app: myapp
```

**Use case:** Prevent all pods from being evicted during node maintenance

### Topology Spread Constraints

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
```

**Result:** Replicas evenly spread across zones (max 1 pod difference per zone)

### Multi-AZ Deployment Checklist
- [ ] Nodes across 3+ AZs
- [ ] PDB for critical workloads
- [ ] Topology spread constraints
- [ ] Multi-AZ database (RDS, managed)
- [ ] HPA configured for load spikes
- [ ] Cluster autoscaler/Karpenter enabled

---

# Security Deep Dive - Study Guide

## Quick Summary (1.5 hours)

### RBAC (Role-Based Access Control)

```yaml
# Role: defines permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]

---
# RoleBinding: assigns role to user/group/serviceaccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- kind: User
  name: alice@example.com
```

**Best Practice:** Create roles per team/app, least privilege

### Pod Security Standards

**Three levels:**
1. **Restricted:** Maximum security (no root, read-only FS, no privileged)
2. **Baseline:** Minimal restrictions (allows common setups)
3. **Privileged:** No restrictions (for system pods)

```yaml
apiVersion: policy/v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsReadOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
  containers:
  - name: app
    image: app:latest
    securityContext:
      capabilities:
        drop:
        - ALL
```

### Secrets Management

```yaml
# Option 1: Kubernetes Secrets (not encrypted by default!)
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
data:
  password: cGFzc3dvcmQxMjM=  # base64 encoded

---
# Option 2: AWS Secrets Manager (via External Secrets Operator)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mysecret
spec:
  secretStoreRef:
    name: aws-sm
    kind: SecretStore
  target:
    name: mysecret
  data:
  - secretKey: password
    remoteRef:
      key: prod/db-password
```

**Best Practice:** Use external secret manager (AWS SM, Azure KV, Vault)

### Image Scanning

```bash
# Trivy
trivy image myapp:latest

# Output:
# CRITICAL CVE-2023-1234 openssl 1.1.1 < 1.1.1w

# Block vulnerable images
# - Run in CI/CD pipeline
# - Use admission webhook (OPA/Gatekeeper)
# - Setup ECR/ACR automatic scanning
```

### Network Policies (Zero-Trust)

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Security Checklist:**
- [ ] RBAC configured (least privilege)
- [ ] Pod security standards enforced
- [ ] Secrets encrypted (external manager)
- [ ] Image scanning in CI/CD
- [ ] Network policies (zero-trust)
- [ ] Regular security audits (kube-bench)

---

# Observability Stack - Study Guide

## Quick Summary (1.5 hours)

### Prometheus + Grafana

```yaml
# Install
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace

# Query (PromQL)
rate(http_requests_total[5m])  # Request rate per second
histogram_quantile(0.99, latency_bucket)  # P99 latency

# Dashboard
# Access: kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

### Logging Stack (ELK or Loki)

**Loki (lightweight):**
```bash
helm install loki grafana/loki-stack -n logging --create-namespace
```

**Query:**
```
{job="kubernetes-pods"} | json | level="error"
```

**Elasticsearch (full-featured):**
```bash
helm install elasticsearch elastic/elasticsearch -n logging
```

### Distributed Tracing (Jaeger)

```bash
helm install jaeger jaegertracing/jaeger -n tracing --create-namespace

# Instrument app (OpenTelemetry)
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("operation"):
    # Your code here
    pass
```

### Observability Checklist
- [ ] Prometheus + Grafana for metrics
- [ ] Loki or ELK for logs
- [ ] Jaeger for tracing
- [ ] AlertManager for alerts
- [ ] 3 pillars connected: metrics → logs → traces

---

# Helm & GitOps - Study Guide

## Quick Summary (1 hour)

### Helm Basics

```bash
# Create chart
helm create myapp

# Deploy
helm install myapp ./myapp

# Upgrade
helm upgrade myapp ./myapp --values prod-values.yaml

# Rollback
helm rollback myapp 1
```

### Helm Templates

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: "1.0"

---
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### GitOps (ArgoCD)

```bash
helm install argocd argo/argo-cd -n argocd --create-namespace
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp
    path: helm/myapp
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Benefits:**
- Source of truth in Git
- Automatic syncing
- Easy rollback (git revert)
- Audit trail (git history)

---

# Hands-On Practicals - Checklist

1. **StatefulSet with PVC:** Deploy PostgreSQL with persistence
2. **NetworkPolicy:** Implement zero-trust network
3. **HPA:** Scale workload based on CPU
4. **Multi-arch:** Build for x64 + ARM64
5. **Backup/Restore:** etcd backup and recovery
6. **Observability Stack:** Deploy Prometheus + Grafana + Loki
7. **Security Scanning:** Trivy + OPA/Gatekeeper

---

# Workspace Study - Key Items

1. **EKS Terraform:** hansen-cloud-native-suite/eksv2/
   - Node groups, add-ons, VPC integration
   
2. **Helm Charts:** hansen-cloud-native-apps/apps/cpq/helm/
   - Deployment templates, HPA setup
   
3. **Observability:** hansen-cloud-native-suite/logging/
   - Jaeger, OpenTelemetry stack

---

# Mock Interview Prep - Sample Scenarios

**Scenario 1:** "Design EKS cluster for startup, 1000 RPS, budget $10K/month"
**Scenario 2:** "Pod can't access S3, troubleshoot step by step"
**Scenario 3:** "Design multi-cloud deployment (AWS + Azure)"
**Scenario 4:** "Cluster performance degraded, debug with 1 hour SLA"

---

## Next Steps

1. Complete all hands-on practicals (each 30-60 min)
2. Review workspace deployments (CPQ, CS charts)
3. Practice mock interview scenarios (2-3 hours)
4. Create personal study notes/cheat sheet
5. Review weak areas before interview

**Total Remaining Time:** 20-25 hours for complete mastery
