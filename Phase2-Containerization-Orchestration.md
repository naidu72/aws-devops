# Phase 2: Containerization & Orchestration

## Study Duration: Days 2-3
## Topics Covered: Docker, ECS, EKS, Kubernetes

---

## 2.1 Docker & Container Concepts

### Core Docker Concepts

#### 1. Docker Image Layers

**Layer Structure:**
- Each line in Dockerfile creates a new layer
- Layers are read-only
- Final layer has write access (container layer)
- Layers are stacked and cached

**Dockerfile Example:**
```dockerfile
# Layer 1: Base image
FROM ubuntu:22.04

# Layer 2: Update packages
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip

# Layer 3: Copy application
COPY app.py /app/app.py

# Layer 4: Set working directory
WORKDIR /app

# Layer 5: Install dependencies
RUN pip install -r requirements.txt

# Layer 6: Set entrypoint
ENTRYPOINT ["python3", "app.py"]
```

**Layer Caching Benefits:**
- Layers reused if unchanged
- Speeds up rebuilds
- Order matters (frequently changing layers last)

**Image Size Optimization:**
```dockerfile
# Bad: Multiple layers, large final image
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y python3 python3-pip
RUN pip install django
RUN apt-get install -y curl wget git
RUN apt-get clean

# Good: Consolidated layers, multi-stage build
FROM ubuntu:22.04 as builder
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    curl \
    wget \
    git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir django

FROM ubuntu:22.04
COPY --from=builder / /
```

#### 2. Dockerfile Best Practices

**1. Use Specific Base Image Tags (not latest):**
```dockerfile
# Bad
FROM ubuntu:latest
FROM node:latest

# Good
FROM ubuntu:22.04
FROM node:20-alpine
```

**2. Minimize Layers:**
```dockerfile
# Bad: 5 layers
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# Good: 1 layer
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    curl \
    git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**3. Use .dockerignore:**
```
.git
.gitignore
node_modules
npm-debug.log
.DS_Store
.env
```

**4. Multi-Stage Build for Production:**
```dockerfile
# Build stage
FROM node:20 as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Final stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY app.js .
USER node
EXPOSE 3000
CMD ["node", "app.js"]
```

Benefits:
- Smaller final image (no build tools)
- No secrets in final layer
- Faster deployment

**5. Use Specific User (not root):**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY --chown=node:node . .
USER node
EXPOSE 3000
CMD ["node", "app.js"]
```

**6. Health Checks:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:5000/health')"

CMD ["python", "app.py"]
```

#### 3. Container Networking

**Bridge Network (Default):**
- Each container gets own IP
- Containers can communicate by IP
- Port mapping: Container:Host

```bash
# Run container with port mapping
docker run -d -p 8080:80 --name web nginx

# Access on host: localhost:8080
```

**Host Network:**
- Container shares host network namespace
- Direct access to host ports
- No port mapping needed

```bash
docker run -d --network host --name web nginx
# Listening on host port 80 directly
```

**Custom Bridge Network:**
- DNS resolution by container name
- Better isolation than default bridge

```bash
# Create network
docker network create myapp

# Run containers on network
docker run -d --network myapp --name db postgres
docker run -d --network myapp --name web nginx

# Within web container: can resolve "db" to database IP
```

**Service Discovery:**
- Containers resolve service names via embedded DNS
- Built-in load balancing in Swarm/Kubernetes
- Service name → VIP → multiple container IPs

#### 4. Volume Mounting & Data Persistence

**Three Mounting Types:**

1. **Volumes:**
   - Managed by Docker
   - Persists after container deletion
   - Mounted at `/var/lib/docker/volumes`
   - Best for databases

```bash
# Create volume
docker volume create mydata

# Run with volume
docker run -d -v mydata:/data postgres

# Data persists even if container deleted
docker rm container_id
docker run -d -v mydata:/data postgres  # Data still there!
```

2. **Bind Mounts:**
   - Mount host directory into container
   - Host controls the path
   - Good for development

```bash
# Mount host directory
docker run -d -v /home/user/code:/app node app.js

# Changes on host reflected in container (watch mode)
# Edit /home/user/code/app.js → container sees change
```

3. **tmpfs:**
   - In-memory storage
   - Deleted when container stops
   - Good for temporary data, sensitive info

```bash
docker run -d --tmpfs /tmp:size=500m nginx
```

**Comparison:**

| Feature | Volumes | Bind Mounts | tmpfs |
|---------|---------|------------|-------|
| **Persistence** | Yes | Yes | No |
| **Performance** | Good | Slower (I/O) | Very fast |
| **Use Case** | DB data | Development | Temp files |
| **Security** | Good | Host access | Memory only |

#### 5. Container Security Scanning

**Multi-Stage Build for Security:**
```dockerfile
# Build stage: includes dev dependencies
FROM python:3.11 as builder
RUN pip install --upgrade pip setuptools
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt

# Final stage: minimal dependencies
FROM python:3.11-slim
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY app.py .
USER nobody
CMD ["python", "app.py"]
```

**Security Best Practices:**
```bash
# Scan image for vulnerabilities
docker scan myimage:latest
# Or use Trivy
trivy image myimage:latest

# Check for latest vulnerabilities
# Pin base image version (never use latest)
# Scan in CI/CD pipeline before pushing
# Use minimal base images (alpine, distroless)
```

**Distroless Images:**
```dockerfile
# Regular: ~300MB with shell, package manager, etc.
FROM python:3.11-slim
RUN pip install flask
COPY app.py .
CMD ["python", "app.py"]

# Distroless: ~200MB, no shell, just runtime
FROM python:3.11-slim as builder
RUN pip install flask
FROM gcr.io/distroless/python3
COPY --from=builder /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY app.py .
ENTRYPOINT ["python3", "app.py"]
```

### Interview Questions Deep Dive

**Q1: What are Docker image layers and why are they important?**

*Expected Answer:*
- Each Dockerfile instruction creates a layer
- Layers are immutable and can be reused
- Caching: If layer unchanged, skip rebuild
- Ordering matters: Put frequent changes last
- Smaller base images = fewer layers to scan
- Each layer adds to final image size

*Demonstrate:*
- Show layering with `docker history myimage`
- Explain cache invalidation scenarios
- Multi-stage builds reduce final size

**Q2: How do you design Dockerfiles for production with security in mind?**

*Expected Answer:*
- Use specific base image tags (not latest)
- Run as non-root user
- Multi-stage builds to exclude build tools
- Minimal base images (alpine, distroless)
- No secrets in Dockerfile
- Health checks for monitoring
- Scan for vulnerabilities in CI/CD

**Q3: Explain the difference between Docker volumes, bind mounts, and tmpfs**

*Expected Answer:*
- Volumes: Managed, persistent, good for production
- Bind mounts: Host path, good for dev
- tmpfs: Memory-only, good for temp data
- Use volumes for database data
- Use bind mounts for local development
- Use tmpfs for secrets, temp files

**Q4: How would you implement container health checks in production?**

*Expected Answer:*
- HEALTHCHECK instruction in Dockerfile
- Configure interval, timeout, retries
- Return 0 for healthy, non-zero for unhealthy
- Orchestrator (Docker Swarm, Kubernetes) monitors health
- Unhealthy containers replaced automatically
- Different probe types: HTTP, TCP, shell command

---

## 2.2 AWS ECS (Elastic Container Service)

### ECS Architecture

#### 1. Task Definitions

**What is a Task Definition:**
- Blueprint for Docker containers
- Specifies image, memory, CPU, ports, volumes
- Similar to docker-compose but AWS-native
- Versioned (v1, v2, v3...)

**Example Task Definition:**
```json
{
  "family": "my-app",
  "containerDefinitions": [
    {
      "name": "my-app-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 0,
          "protocol": "tcp"
        }
      ],
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "environment": [
        {
          "name": "LOG_LEVEL",
          "value": "INFO"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      }
    }
  ],
  "requiresCompatibilities": ["EC2"],
  "cpu": "256",
  "memory": "512"
}
```

#### 2. ECS vs Fargate Launch Types

**EC2 Launch Type:**
- You manage EC2 instances
- Cluster = collection of EC2 instances
- Tasks run on instances in cluster
- Cheaper for sustained workloads
- Full control over infrastructure

```
Task → ECS Container Agent → EC2 Instance → VPC
```

**Fargate Launch Type:**
- AWS manages infrastructure
- No EC2 instances to provision
- Pay per task per hour
- Better for sporadic/variable workloads
- Requires VPC/subnets configuration

```
Task → Fargate → Managed by AWS
```

**When to Use:**

| Use Case | EC2 | Fargate |
|----------|-----|--------|
| **Sustained load** | ✓ | |
| **Variable load** | | ✓ |
| **Cost sensitive** | ✓ | |
| **Compliance** | ✓ | |
| **DevOps heavy** | | ✓ |
| **Always running** | ✓ | |

#### 3. ECS Services

**Service = Desired State Management**
- Ensures N replicas running
- Replaced unhealthy tasks
- Integrated with load balancer
- Auto-scaling policies

```bash
# Example: Ensure 3 replicas of web app always running
aws ecs create-service \
  --cluster my-cluster \
  --service-name web-service \
  --task-definition my-app:1 \
  --desired-count 3 \
  --launch-type EC2 \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=my-app-container,containerPort=8080
```

**Service Auto-Scaling:**
```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/web-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 10

# Create scaling policy (target tracking)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/web-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 300
  }'
```

#### 4. Task Placement Strategies (EC2)

**How ECS Chooses Which EC2 to Run Task:**

1. **Spread:**
   - Distribute across instances
   - Maximize availability
   - If instance fails, only 1 task affected

2. **Binpack:**
   - Pack tasks tightly on instances
   - Minimize number of instances
   - Cost optimized (fewer running instances)

3. **Random:**
   - Random placement
   - Simple but less predictable

```bash
# Create service with spread strategy
aws ecs create-service \
  --cluster my-cluster \
  --service-name web-service \
  --task-definition my-app:1 \
  --desired-count 3 \
  --placement-strategy \
    type=spread,field=instanceId \
    type=spread,field=attribute:ecs.availability-zone
```

#### 5. Container Insights & Monitoring

**CloudWatch Container Insights:**
- Metrics for cluster, service, task
- CPU, memory, network metrics
- Task-level granularity
- Dashboard visualizations

```bash
# Enable Container Insights on cluster
aws ecs put-cluster-settings \
  --cluster my-cluster \
  --settings name=containerInsights,value=enabled

# View Container Insights metrics
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name TaskCount \
  --dimensions Name=ClusterName,Value=my-cluster Name=ServiceName,Value=web-service \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --statistics Average
```

#### 6. Secrets Management in ECS

**Option 1: Secrets Manager**
```bash
# Create secret
aws secretsmanager create-secret \
  --name prod/db/password \
  --secret-string '{"username":"admin","password":"MyPassword123"}'

# Reference in task definition
{
  "name": "DB_PASSWORD",
  "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db/password"
}
```

**Option 2: Parameter Store**
```bash
# Create parameter
aws ssm put-parameter \
  --name /prod/app/log_level \
  --value INFO \
  --type String

# Reference in task definition
{
  "name": "LOG_LEVEL",
  "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/prod/app/log_level"
}
```

### Interview Questions Deep Dive

**Q1: Explain the difference between EC2 and Fargate launch types in ECS**

*Expected Answer:*
- EC2: You manage instances, more control, cheaper at scale
- Fargate: AWS manages, simpler, better for variable loads
- EC2: Best for predictable, sustained workloads
- Fargate: Best for variable, serverless-like workloads
- Cost: EC2 cheaper per task but requires instance capacity
- Fargate: Simpler ops, no server management

**Q2: How would you troubleshoot ECS task failures?**

*Expected Answer:*
- Check task status and stop code (CannotInspectContainerError, etc.)
- View container logs (CloudWatch Logs)
- Check task definition: memory, CPU, environment variables
- Verify IAM permissions (task role, execution role)
- Check security groups allow traffic
- Run task locally to reproduce issue
- Container exit code indicates what failed

**Q3: Design a highly available ECS service architecture**

*Expected Answer:*
- Multi-AZ deployment (subnets in multiple AZs)
- ALB with health checks
- Auto-scaling based on metrics
- Spread placement strategy
- Container Insights monitoring
- Proper health checks in task definition
- Sufficient instance capacity

**Q4: How do you handle secrets management in ECS?**

*Expected Answer:*
- Never hardcode in task definition
- Use Secrets Manager or Parameter Store
- Task execution role needs permission to retrieve
- Reference by ARN in task definition
- Rotate secrets periodically
- Audit access in CloudTrail

---

## 2.3 AWS EKS & Kubernetes Fundamentals

### Kubernetes Architecture

#### 1. Cluster Components

**Control Plane (Managed by AWS on EKS):**
- API Server: REST API, state management
- etcd: Distributed key-value store (cluster state)
- Scheduler: Assigns pods to nodes
- Controller Manager: Runs controllers (replica, job, etc.)
- Cloud Controller Manager: AWS resource integration

**Worker Nodes:**
- kubelet: Agent ensuring pods running
- kube-proxy: Network proxy, service routing
- Container runtime: Docker, containerd, etc.

**EKS Specifics:**
- Control plane in AWS-managed account
- You manage worker nodes (EC2, Fargate)
- AWS handles etcd, API server, scheduler
- You only interact with API server

#### 2. Kubernetes Objects

**Pod:**
- Smallest deployable unit
- Can contain multiple containers (rarely)
- Shared network namespace (localhost accessible)
- Ephemeral (deleted and recreated)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
```

**Deployment:**
- Manages replica sets
- Rolling updates
- Rollback capability
- Desired state specified

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
```

**Service:**
- Stable endpoint for pods
- Load balancing across pods
- DNS resolution
- Three types: ClusterIP, NodePort, LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**StatefulSet:**
- For stateful applications
- Stable pod identities (nginx-0, nginx-1)
- Ordered pod creation/deletion
- Persistent storage per pod

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
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
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

#### 3. Service Discovery & Ingress

**Service Discovery (Internal):**
- Service gets DNS name: `nginx-service.default.svc.cluster.local`
- Pods resolve to service IP
- Service IP routes to pod IPs (kube-proxy)

**Ingress (External):**
- HTTP(S) routing rules
- External IP/DNS access
- AWS ALB Ingress Controller needed on EKS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

#### 4. Persistent Volumes & Storage

**Persistent Volume (PV):**
- Cluster-level storage resource
- Lifecycle independent of pod

**Persistent Volume Claim (PVC):**
- Storage request by pod
- Binds to PV automatically

**EBS Storage Class on EKS:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: ebs-gp3
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

#### 5. RBAC (Role-Based Access Control)

**Service Account:**
- Identity for pods
- Token for API authentication

**Role:**
- Set of permissions
- Namespace-scoped

**RoleBinding:**
- Binds role to service account

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["app-secret"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-reader
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: default
```

#### 6. Network Policies

**Network Policy:**
- Firewall rules for pod-to-pod communication
- Ingress and egress rules

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

#### 7. Helm Package Manager

**Helm Chart:**
- Template package for Kubernetes
- Reusable across clusters
- Similar to package management (apt, yum)

**Structure:**
```
my-chart/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
```

**Usage:**
```bash
# Create chart
helm create my-app

# Install
helm install my-release my-app -f values.yaml

# Upgrade
helm upgrade my-release my-app --values new-values.yaml

# Rollback
helm rollback my-release 1
```

#### 8. Cluster Autoscaling

**Cluster Autoscaler:**
- Adds/removes nodes based on pod demand
- Scale up: When pod pending (insufficient resources)
- Scale down: When nodes underutilized

```bash
# Install Cluster Autoscaler on EKS
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --values autoscaler-values.yaml
```

### Interview Questions Deep Dive

**Q1: Explain the difference between Deployment and StatefulSet**

*Expected Answer:*
- Deployment: Stateless applications, pod replicas identical
- StatefulSet: Stateful applications, stable pod identities
- Deployment: Pods can be replaced in any order
- StatefulSet: Pods created/deleted in order (nginx-0, 1, 2)
- Deployment: Services with randomized pod selection
- StatefulSet: Persistent storage per pod
- Use Deployment for web servers, microservices
- Use StatefulSet for databases, cache clusters

**Q2: How would you implement multi-tenancy in a shared Kubernetes cluster?**

*Expected Answer:*
- Namespace isolation (logical separation)
- RBAC to restrict permissions per tenant
- Network Policies for pod-to-pod isolation
- Resource quotas per namespace
- Service accounts per tenant
- Monitoring/billing per namespace
- Different storage classes per tenant if needed

**Q3: Design a Kubernetes cluster for high availability and disaster recovery**

*Expected Answer:*
- Multi-AZ node distribution
- Anti-affinity rules spread pods across nodes
- Persistent volumes with backup strategy
- Redundant control plane (managed by EKS)
- Backup etcd periodically
- Test recovery procedures
- Multi-region for geographic DR

**Q4: How do you manage secrets securely in Kubernetes?**

*Expected Answer:*
- Use Kubernetes Secrets (base64 encoded in etcd)
- Better: Use external secret management (AWS Secrets Manager)
- Even better: Use secret management operator (Sealed Secrets, External Secrets)
- Restrict RBAC access to secrets
- Encrypt etcd at rest (AWS EKS supports this)
- Use IAM for workload identity (IRSA)
- Never commit secrets to Git

**Q5: Explain ingress controllers and their role in routing**

*Expected Answer:*
- Ingress resource defines routing rules
- Ingress Controller implements routing (ALB, NGINX, etc.)
- On EKS: AWS ALB Ingress Controller
- Creates AWS ALB for Ingress resources
- Handles path-based and host-based routing
- SSL/TLS termination
- DDoS protection integration

---

## Kubernetes vs ECS Comparison

| Feature | Kubernetes | ECS |
|---------|---|---|
| **Learning Curve** | Steep | Moderate |
| **Community** | Very large | Large |
| **Features** | More advanced | Simpler |
| **Multi-cloud** | Yes | AWS only |
| **Stateful Apps** | Better (StatefulSet) | Possible |
| **Ecosystem** | Huge (Helm, etc.) | Smaller |
| **Ops Complexity** | Higher | Lower |

---

## Key Concepts Checklist

- [ ] Docker layers and caching strategy
- [ ] Multi-stage builds for optimization
- [ ] Container security best practices
- [ ] ECS task definitions and services
- [ ] EC2 vs Fargate trade-offs
- [ ] ECS service auto-scaling
- [ ] Kubernetes deployment manifests
- [ ] StatefulSet vs Deployment
- [ ] Service discovery in Kubernetes
- [ ] Persistent volumes and storage
- [ ] RBAC and service accounts
- [ ] Network policies
- [ ] Helm basics
- [ ] Cluster autoscaling
- [ ] Ingress controllers

---

**Total Phase 2 Study Time: 5-6 hours**
