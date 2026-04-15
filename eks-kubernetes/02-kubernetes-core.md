# 02 - Kubernetes Core Concepts: Workloads and Resources

## Kubernetes Workload Types

### Deployments (Most Common)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      version: v1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 pod more than desired
      maxUnavailable: 0   # 0 pods can be unavailable
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - payment-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: payment-service
        image: myregistry.azurecr.io/payment-service:v1.2.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-creds
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
```

**Why Deployments?**
- Automatically manages ReplicaSet
- Rolling updates with zero downtime
- Rollback capability
- Scales to any number of replicas

**What happens during update:**
```
replicas: 3 (existing)
maxSurge: 1
maxUnavailable: 0

Step 1: Launch new pod with v1.2 (now 4 pods)
Step 2: Pod v1.2 ready, terminate old pod (3 pods)
Step 3: Launch new pod with v1.2 (4 pods)
Step 4: Pod v1.2 ready, terminate old pod (3 pods)
Step 5: Launch new pod with v1.2 (4 pods)
Step 6: Pod v1.2 ready, terminate old pod (3 pods)

Result: 3 pods with v1.2, zero downtime!
```

---

### StatefulSets (Databases, Message Queues)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-primary
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ebs-gp3
      resources:
        requests:
          storage: 100Gi
```

**Why StatefulSets?**
- Persistent storage with PersistentVolumeClaim
- Stable pod identity (postgres-0, postgres-1)
- Ordered pod startup and shutdown
- Used for: databases, Kafka, Elasticsearch

**Key differences from Deployment:**
```
Deployment: Interchangeable pods
StatefulSet: Unique pod identities

Deployment uses random names: payment-abc123
StatefulSet uses ordinal names: postgres-0, postgres-1, postgres-2

Deployment pods: Can be deleted in any order
StatefulSet pods: Deleted in reverse order (postgres-2, then postgres-1, then postgres-0)
```

---

### DaemonSets (Run on Every Node)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

**Used for:**
- Monitoring agents (Prometheus node-exporter)
- Log collectors (Fluentd, CloudWatch agent)
- Security agents (Falco)
- Network plugins (Calico, Weave)

**Key feature:** One pod per node, automatically!

---

## Services: Exposing Workloads

### ClusterIP (Internal Communication)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
spec:
  type: ClusterIP
  clusterIP: 10.100.0.42  # Assigned by Kubernetes
  selector:
    app: payment-service
  ports:
  - name: http
    port: 80           # Service port
    targetPort: 8080   # Pod port
    protocol: TCP
```

**How it works:**
```
Pod A wants to reach payment-service
    ↓
DNS: payment-service.production.svc.cluster.local = 10.100.0.42
    ↓
kube-proxy creates iptables rule:
  traffic to 10.100.0.42:80 → DNAT to pod IP:8080
    ↓
Load balanced across all pods with label "app: payment-service"
    ↓
Request reaches Pod B, Pod C, or Pod D randomly
```

**Why ClusterIP?**
- No external exposure
- Internal cluster-only communication
- Stable DNS name (independent of pod IPs)
- Round-robin load balancing

---

### LoadBalancer (External Access via AWS ALB/NLB)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: api-gateway
  ports:
  - name: https
    port: 443
    targetPort: 8080
    protocol: TCP
  sessionAffinity: ClientIP
```

**What happens:**
```
1. Service created with type: LoadBalancer
2. AWS Load Balancer Controller detects it
3. Creates Network Load Balancer (NLB) in AWS
4. Configures health checks pointing to pods
5. External users access: nlb-123.us-east-1.elb.amazonaws.com
6. NLB routes traffic to pods on each node
```

**Cost consideration:**
- Each LoadBalancer service = 1 AWS Load Balancer (~$18-25/month)
- Use LoadBalancer sparingly (usually 1-2 services)
- For many services, use Ingress instead

---

### Ingress (Layer 7 - HTTP/HTTPS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /v1/payments
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

**What happens:**
```
1. Ingress created with 2 hostnames
2. AWS Load Balancer Controller creates Application Load Balancer (ALB)
3. ALB rules:
   - api.example.com/v1/users → user-service:80
   - api.example.com/v1/payments → payment-service:80
   - admin.example.com/* → admin-service:80
4. Path-based and host-based routing at Layer 7
5. Expensive resources (but cheaper than multiple LoadBalancer services)
```

**Use Ingress when:**
- Multiple services need external access
- Want path-based or host-based routing
- Need SSL/TLS termination
- Cost-effective (1 ALB for multiple services)

---

## Resource Management: Requests and Limits

### QoS (Quality of Service) Classes

```yaml
# Guaranteed QoS (requests = limits)
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 512Mi
# Pod will NOT be evicted (highest priority)

---

# Burstable QoS (requests < limits)
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
# Pod CAN be evicted if node under pressure
# Can use more than requested (up to limit)

---

# BestEffort QoS (no requests or limits)
# Pod CAN be evicted first
# Not recommended for production!
```

### Memory Pressure Scenario

```
Node: 16 GB total memory

Running pods:
- Pod A (Guaranteed): 512Mi requested, 512Mi limit
- Pod B (Burstable): 1Gi requested, 2Gi limit, using 1.5Gi
- Pod C (BestEffort): no requests, no limits, using 3Gi

New Pod D arrives (Guaranteed): 512Mi requested

Memory pressure: Node has only 8Gi free, need 512Mi

Kubernetes eviction order (lowest priority first):
1. Pod C (BestEffort) - EVICTED! (didn't specify any requests)
2. Pod B (Burstable) - EVICTED! (using more than requested)
3. Pod A (Guaranteed) - SAFE! (using exactly what requested)
4. Pod D (Guaranteed) - Scheduled successfully

Result: Pod C and B are killed to make room for Pod D
```

**Production rule:**
```
Always set requests and limits for critical services!
- Set limits slightly above average usage to prevent eviction
- Set requests = or slightly below average usage
- Monitor actual usage: kubectl top pod -A
```

---

## Pod Disruption Budgets (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: production
spec:
  minAvailable: 2        # At least 2 pods must be available
  selector:
    matchLabels:
      app: payment-service
```

**When does PDB matter?**

```
Scenario: Cluster needs to update nodes (voluntary disruption)

Without PDB:
- Kubernetes kills all payment-service pods on the node
- If 3 pods → all 3 killed during node drain
- Service completely down!

With PDB (minAvailable: 2):
- Kubernetes kills only 1 pod to maintain 2 available
- Other replicas still serving traffic
- Zero downtime!

PDB prevents:
1. Node draining (updates, scaling down)
2. Pod eviction (resource pressure)
3. Intentional disruptions

PDB does NOT prevent:
1. Pod crash (app error)
2. Node failure (hardware failure)
3. Forced pod deletion (kubectl delete --force)
```

---

## Namespaces and Resource Quotas

### Namespace Isolation

```yaml
# Create namespace
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    env: production
    team: payments

---

# ResourceQuota: Limit total resources per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "10"        # Total CPU requests in namespace
    requests.memory: "20Gi"   # Total memory requests
    limits.cpu: "20"          # Total CPU limits
    limits.memory: "40Gi"     # Total memory limits
    pods: "100"               # Max pods in namespace
    services: "10"

---

# NetworkPolicy: Restrict traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-deny-all
  namespace: team-payments
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: team-payments
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: team-payments
    ports:
    - protocol: TCP
      port: 5432  # Database
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53  # DNS
```

**Why namespaces?**
- Multi-tenancy: Different teams in same cluster
- Environment separation: dev, staging, prod
- Resource quotas: Prevent one service from consuming all resources
- RBAC: Different permissions per namespace

---

## RBAC: Role-Based Access Control

```yaml
# Define role: what actions are allowed
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-reader
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch"]

---

# Bind role to user/service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-deployments
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployment-reader
subjects:
- kind: User
  name: "jane.doe@company.com"
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: ci-cd-bot
  namespace: production

---

# Service account for pods
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-deployer
  namespace: production

---

# Bind to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-deployer-role
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployment-reader
subjects:
- kind: ServiceAccount
  name: app-deployer
  namespace: production
```

**Common verbs:**
```
get       - Get a single resource
list      - List all resources
watch     - Watch for changes
create    - Create new resource
update    - Update existing resource
patch     - Partially update resource
delete    - Delete resource
exec      - Execute command in pod
logs      - Read pod logs
port-forward - Forward ports
```

---

## ConfigMaps and Secrets

### ConfigMap (Non-sensitive configuration)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_POOL_SIZE: "20"
  LOG_LEVEL: "INFO"
  FEATURES_ENABLED: "payment-v2,analytics"
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://backend:8080;
        }
    }
```

**Mount in pod:**
```yaml
containers:
- name: app
  image: myapp:latest
  env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
  volumeMounts:
  - name: config
    mountPath: /etc/nginx
volumes:
- name: config
  configMap:
    name: app-config
```

### Secret (Sensitive data - BASE64 encoded)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: cG9zdGdyZXM=        # base64: postgres
  password: c3VwZXJzZWNyZXQ=    # base64: supersecret
  connection-string: cG9zdGdyZXM6Ly9sb2NhbGhvc3Q=  # base64: encoded URL
```

**Mount in pod:**
```yaml
containers:
- name: app
  env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

**Security note:**
```
base64 is NOT encryption! It's just encoding.
Anyone who decodes sees the password.

Real security requires:
1. Encrypt secrets at rest (etcd encryption)
2. RBAC to restrict who can read secrets
3. Use Secrets Manager (AWS) with IRSA
4. Scan secret repos for accidental commits
```

---

## Labels and Selectors

### Why Labels?

```yaml
# Organization and selection
labels:
  app: payment-service
  version: v1.2.0
  environment: production
  team: platform-eng
  cost-center: "9999"

---

# Selector: Query by labels
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
    environment: production
  ports:
  - port: 80
    targetPort: 8080

# This service selects ALL pods with:
# app: payment-service AND environment: production
```

**Common labels:**
```
app=           - Application name
version=       - Version
environment=   - dev, staging, prod
tier=          - frontend, backend, database
team=          - Which team owns it
cost-center=   - For billing
release=       - Release name (for Helm)
```

---

## Interview Q&A: Kubernetes Core

### Q: "What's the difference between Deployment and StatefulSet?"

**Answer:**
"Deployments are for stateless workloads where pods are interchangeable. They manage replicas and rolling updates. Each pod gets a random name. StatefulSets are for stateful workloads like databases. They give pods stable, ordinal identities (postgres-0, postgres-1). StatefulSets manage PersistentVolumeClaims, so each pod gets its own storage. For my payment microservice, I use Deployment because it's stateless. For the database, I use StatefulSet because it needs persistent storage and stable identity."

### Q: "How do resource requests and limits affect scheduling?"

**Answer:**
"Requests are used by the scheduler to decide which node a pod can fit on. If a node has 4 CPU free and a pod requests 1 CPU, it's scheduled there. Limits prevent a pod from using more than that amount. If a pod tries to exceed its memory limit, Kubernetes kills it. For production, I always set requests = expected usage and limits = peak usage with some headroom. This ensures the scheduler makes good decisions and prevents noisy neighbor problems."

### Q: "What are Pod Disruption Budgets?"

**Answer:**
"PDBs ensure minimum availability during voluntary disruptions like node drains. If I have 3 payment-service pods and set minAvailable=2, Kubernetes will evict only 1 pod at a time during node updates, maintaining 2 available. This prevents downtime during cluster maintenance. PDBs don't protect against involuntary disruptions like pod crashes."

