# Integration Scenarios: Bringing It All Together

This guide shows how all DevOps components work together in real-world scenarios.

---

## Scenario 1: Deploy Payment Microservice End-to-End

### Overview
Deploy a new payment processing microservice from code commit to production with monitoring, canary deployment, and automatic rollback capability.

### Architecture

```
Developer Code Push
    ↓
GitHub → GitHub Actions
    ↓ (Build)
Docker Image → ECR
    ↓ (Deploy)
Terraform → EKS Cluster
    ↓ (Traffic Management)
Istio Service Mesh
    ↓ (Health Check)
Prometheus + CloudWatch
```

### Step-by-Step Walkthrough

#### Phase 1: Infrastructure Setup (Terraform)

```hcl
# 1. Define EKS cluster across 3 AZs
module "eks_cluster" {
  source = "./modules/eks"
  cluster_name = "prod-payment"
  node_count = 6
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# 2. Define VPC with private subnets
module "vpc" {
  source = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# 3. Define IAM roles for pods (IRSA)
module "irsa" {
  source = "./modules/irsa"
  service_account_name = "payment-sa"
  oidc_provider = module.eks_cluster.oidc_provider
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "secretsmanager:GetSecretValue"
      ]
      Resource = "*"
    }]
  })
}

# 4. Define ECR repository
resource "aws_ecr_repository" "payment_service" {
  name                 = "payment-service"
  image_tag_mutability = "IMMUTABLE"
  image_scanning_configuration {
    scan_on_push = true
  }
}
```

**Terraform Apply:**
```bash
terraform plan -out=tfplan
terraform apply tfplan
# Result: EKS cluster, VPC, IAM roles, ECR repository ready
```

#### Phase 2: Build and Publish (CI/CD)

Developer pushes code:
```bash
git push origin feature/payment-processing
```

GitHub Actions workflow triggers:

```yaml
name: Build and Deploy Payment Service

on:
  push:
    branches: [main, develop]
    paths: ['payment-service/**']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    # 1. Build Docker image
    - name: Build Docker image
      run: |
        docker build -t payment-service:${{ github.sha }} \
          --cache-from payment-service:latest \
          -f payment-service/Dockerfile \
          payment-service/
    
    # 2. Scan for vulnerabilities
    - name: Scan image with Trivy
      run: |
        trivy image payment-service:${{ github.sha }}
    
    # 3. Run tests
    - name: Run unit tests
      run: |
        docker run payment-service:${{ github.sha }} go test ./...
    
    # 4. Push to ECR
    - name: Push to ECR
      run: |
        aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com
        docker tag payment-service:${{ github.sha }} ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/payment-service:${{ github.sha }}
        docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/payment-service:${{ github.sha }}
    
    # 5. Deploy to dev
    - name: Deploy to DEV
      if: github.ref == 'refs/heads/develop'
      run: |
        helm upgrade payment-service ./helm/payment-service \
          -n dev \
          -f helm/values-dev.yaml \
          --set image.tag=${{ github.sha }}
    
    # 6. Deploy to production (manual approval)
    - name: Deploy to PROD
      if: github.ref == 'refs/heads/main'
      environment: production
      run: |
        helm upgrade payment-service ./helm/payment-service \
          -n production \
          -f helm/values-prod.yaml \
          --set image.tag=${{ github.sha }} \
          --set-file istio.canary.enabled=true
```

**Result:** Image built, tested, scanned, and pushed to ECR. Dev environment auto-deployed.

#### Phase 3: Deploy to Kubernetes (Helm)

```bash
helm upgrade payment-service ./helm/payment-service \
  -n production \
  -f helm/values-prod.yaml \
  --set image.tag=abc123def456
```

**Helm chart structure:**
```
helm/payment-service/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml        # Deployment for payment-service
│   ├── service.yaml           # ClusterIP Service
│   ├── configmap.yaml         # Configuration
│   ├── secret.yaml            # Database credentials
│   ├── hpa.yaml               # Horizontal Pod Autoscaler
│   ├── pdb.yaml               # Pod Disruption Budget
│   ├── virtualservice.yaml    # Istio routing
│   └── destinationrule.yaml   # Istio subsets
```

**Deployment YAML** (from Helm template):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-v2
  namespace: production
spec:
  replicas: 1    # 1 canary pod initially
  selector:
    matchLabels:
      app: payment-service
      version: v2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: payment-service
        version: v2
    spec:
      serviceAccountName: payment-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: payment-service
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/payment-service:abc123def456
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: payment-db-creds
              key: url
        - name: LOG_LEVEL
          value: "INFO"
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 2
        volumeMounts:
        - name: config
          mountPath: /etc/payment-service
      volumes:
      - name: config
        configMap:
          name: payment-service-config
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
```

**Result:** New pods running payment-service v2. Old v1 pods still running.

#### Phase 4: Istio Canary Deployment

**VirtualService** - Defines traffic routing:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
  - payment-service.production.svc.cluster.local
  http:
  - route:
    - destination:
        host: payment-service.production.svc.cluster.local
        subset: v1
      weight: 90
    - destination:
        host: payment-service.production.svc.cluster.local
        subset: v2-canary
      weight: 10
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
```

**DestinationRule** - Defines subsets and traffic policies:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: production
spec:
  host: payment-service.production.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 10
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2-canary
    labels:
      version: v2
```

**Traffic flow:**
```
External request to payment-service
    ↓
Kubernetes DNS resolves payment-service → 10.100.0.42 (ClusterIP)
    ↓
Envoy proxy intercepts traffic
    ↓
Checks VirtualService rules
    ↓
Routes: 90% → v1 pods, 10% → v2 canary pods
    ↓
Checks DestinationRule outlier detection
    ↓
Applies ROUND_ROBIN load balancing
    ↓
Request reaches pod (v1 or v2)
    ↓
Response → Back to client
```

#### Phase 5: Observability & Monitoring

**OpenTelemetry instrumentation** (in payment-service code):

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp"
    "go.opentelemetry.io/otel/sdk/metric"
)

// Initialize OpenTelemetry
exporter, _ := otlpmetrichttp.New(ctx, 
    otlpmetrichttp.WithEndpoint("localhost:4318"))
provider := metric.NewMeterProvider(metric.WithReader(metric.NewPeriodicReader(exporter)))
otel.SetMeterProvider(provider)

// Record metrics
meter := otel.Meter("payment-service")
requestCounter, _ := meter.Int64Counter("payment.requests.total")
requestLatency, _ := meter.Float64Histogram("payment.requests.duration_ms")

// When handling request
requestCounter.Add(ctx, 1, 
    attribute.String("method", "transfer"),
    attribute.String("status", "success"))
requestLatency.Record(ctx, duration,
    attribute.String("method", "transfer"))
```

**Prometheus scraping** (from `/metrics` endpoint):

```
# Payment service metrics
payment_requests_total{method="transfer",status="success"} 1250
payment_requests_total{method="transfer",status="failure"} 3
payment_requests_duration_ms_bucket{method="transfer",le="100"} 1000
payment_requests_duration_ms_bucket{method="transfer",le="500"} 1245
payment_requests_duration_ms_bucket{method="transfer",le="1000"} 1250

# Kubernetes metrics
container_memory_usage_bytes{pod="payment-service-v1-abc123"} 512000000
container_cpu_usage_seconds_total{pod="payment-service-v1-abc123"} 150.5
```

**Grafana dashboard** shows:

```
Canary Deployment Status:
├── v1 (stable): 90% traffic, 1250 req/min, 95ms latency, 99.76% success
├── v2 (canary): 10% traffic, 139 req/min, 98ms latency, 99.78% success
├── Error rate: v1: 0.24%, v2: 0.22% (green ✓)
├── P99 latency: v1: 450ms, v2: 480ms (acceptable)
└── Health: Both versions healthy
```

**CloudWatch alarms** configured for:
- Error rate > 1%
- Latency P99 > 1 second
- Pod crash loops
- Node capacity critical

#### Phase 6: Canary Traffic Shift

**After 1 hour of monitoring** (all metrics healthy):

```bash
# Shift traffic to 25/75
kubectl patch vs payment-service -n production --type merge -p \
'{"spec":{"http":[{"route":[
  {"destination":{"host":"payment-service","subset":"v1"},"weight":75},
  {"destination":{"host":"payment-service","subset":"v2-canary"},"weight":25}
]}]}}'

# After 1 hour at 25/75 (if still healthy)
# Shift traffic to 50/50
# After 1 hour at 50/50 (if still healthy)
# Shift traffic to 75/25
# After 1 hour at 75/25 (if still healthy)
# Shift traffic to 0/100 (v2 fully promoted)
```

**If issues detected at any point:**

```bash
# Immediate rollback to 100/0
kubectl patch vs payment-service -n production --type merge -p \
'{"spec":{"http":[{"route":[
  {"destination":{"host":"payment-service","subset":"v1"},"weight":100},
  {"destination":{"host":"payment-service","subset":"v2-canary"},"weight":0}
]}]}}'

# Investigate issue
kubectl logs deployment/payment-service-v2 -n production
kubectl describe pod payment-service-v2-xyz -n production

# Once issue fixed, can retry or rollback deployment
helm rollback payment-service -n production
```

#### Phase 7: Cleanup and Finalization

**After successful canary promotion**:

```bash
# Delete old v1 deployment
kubectl delete deployment payment-service-v1 -n production

# Verify traffic is 100% on v2
kubectl get vs payment-service -n production

# Update permanent Helm values
# Change: image.tag from old to new
# Change: replicaCount from 1 (canary) to 3 (stable)

helm upgrade payment-service ./helm/payment-service \
  -n production \
  -f helm/values-prod.yaml \
  --set image.tag=v2 \
  --set replicaCount=3
```

---

## Scenario 2: Multi-Environment Multi-Account Deployment

### Use Terragrunt for DRY Infrastructure as Code

**Directory structure:**

```
infrastructure/
├── terragrunt.hcl                 # Root config
├── modules/
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── vpc/
│   │   └── ...
│   └── iam/
│       └── ...
├── dev/
│   ├── terragrunt.hcl            # Dev-specific config
│   └── terraform.tfvars          # Dev values
├── qa/
│   ├── terragrunt.hcl            # QA-specific config
│   └── terraform.tfvars          # QA values
├── prod/
│   ├── terragrunt.hcl            # Prod-specific config
│   └── terraform.tfvars          # Prod values
└── global/
    └── s3-backend.tf             # Shared state backend
```

**Root terragrunt.hcl:**

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "company-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

locals {
  environment = run_cmd("echo $ENVIRONMENT")
}
```

**dev/terragrunt.hcl:**

```hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "${get_parent_terragrunt_dir()}/modules//eks"
}

inputs = {
  cluster_name    = "dev-payment"
  node_count      = 1
  node_type       = "t3.medium"
  environment     = "dev"
}
```

**prod/terragrunt.hcl:**

```hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "${get_parent_terragrunt_dir()}/modules//eks"
}

inputs = {
  cluster_name    = "prod-payment"
  node_count      = 6
  node_type       = "c5.xlarge"
  environment     = "prod"
  multi_az        = true
  auto_scaling    = {
    min_size = 6
    max_size = 12
  }
}
```

**Deploy to all environments:**

```bash
# Deploy dev
cd infrastructure/dev
terragrunt plan
terragrunt apply

# Deploy qa
cd ../qa
terragrunt plan
terragrunt apply

# Deploy prod
cd ../prod
terragrunt plan
terragrunt apply

# Or deploy all at once
cd infrastructure
terragrunt run-all plan
terragrunt run-all apply
```

---

## Scenario 3: Handle Production Incident (Debugging)

### Alert: High Error Rate Detected

**Step 1: CloudWatch Alert Triggers**

```
CloudWatch Alert: error_rate > 1%
├── Current error rate: 3.5%
├── Duration: 5 minutes
├── Affected service: payment-service
├── Time: 2024-04-15 14:32 UTC
```

**Step 2: Check Observability**

Check OpenTelemetry traces:
```
Recent requests:
├── Request 1: 450ms duration, status 500 (error)
├── Request 2: 480ms duration, status 500 (error)
├── Request 3: 5200ms duration, status 504 (timeout)
└── Trace shows: payment-service → database → slow query

Slow query detected:
- SELECT * FROM transactions WHERE ... FULL SCAN
- Should use index
- No WHERE clause on indexed column
```

**Step 3: Check Kubernetes**

```bash
# Check pod status
kubectl get pods -n production -l app=payment-service

# Output:
# NAME                                  READY   STATUS    RESTARTS
# payment-service-v2-abc123xyz          1/2     NotReady  0
# payment-service-v2-def456uvw          1/2     NotReady  0
# payment-service-v2-ghi789rst          1/2     NotReady  0

# Check pod readiness
kubectl describe pod payment-service-v2-abc123xyz -n production

# Shows:
# Readiness probe failed after 5 consecutive failures
# Exec form: GET /health/ready HTTP/1.1
# All pods reporting 503 Service Unavailable
```

**Step 4: Check Logs**

```bash
kubectl logs deployment/payment-service-v2 -n production

# Logs show:
# [ERROR] database connection failed: Connection timeout
# [ERROR] failed to execute query: Connection refused
# [WARN] retrying connection...
# [ERROR] max retries exceeded
```

**Step 5: Investigate Root Cause**

```bash
# Check database connectivity
kubectl exec -it payment-service-v2-abc123xyz -n production -- \
  psql -h db.example.com -U appuser -d payments -c "SELECT 1"

# Output:
# psql: connection refused

# Check Secrets Manager for credentials
aws secretsmanager get-secret-value --secret-id payment-db-creds

# Credentials last updated: 30 minutes ago
# By: automatic rotation
# Check rotation policy... FOUND: Automatic rotation attempted
# Rotation status: FAILED
```

**Root Cause Found:** Database automatic credential rotation failed. Pods have old credentials.

**Step 6: Immediate Mitigation**

```bash
# Option 1: Rollback the deployment
helm rollback payment-service -n production

# This restarts pods with previous deployment
# Previous credentials still in Secrets Manager
# Service recovers within 1-2 minutes

# Result: Error rate drops to 0.1%
# Service back to normal
```

**Step 7: Investigate Root Cause**

```bash
# Check what changed 30 minutes ago
git log --oneline -n 10
helm history payment-service -n production

# Shows: 
# Version 3 (current): Just deployed 35 minutes ago
# Version 2: Previous version
# 
# Changes in v3:
# - New database schema migration
# - Updated connection pool settings
# - Changed password validation

# Database credential rotation happens every 30 minutes
# v3 deployment has stricter password validation
# Rotation creates new password, but password doesn't match validation rules
# Rotation fails
# Pods fail readiness checks
```

**Step 8: Permanent Fix**

```hcl
# Update Terraform for password validation rule
resource "aws_secretsmanager_secret_rotation" "payment_db" {
  secret_id       = aws_secretsmanager_secret.payment_db.id
  rotation_rules {
    automatically_after_days = 30
  }

  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn
  depends_on          = [aws_secretsmanager_secret_version.rotation_lambda_permission]
}

# Update Lambda rotation function to use weaker password validation
# OR update payment-service to accept the new password format
```

```yaml
# Update Helm values
# helm/values-prod.yaml
database:
  passwordValidation:
    pattern: "^[A-Za-z0-9!@#$%^&*()-_=+{}\\[\\]:;'\"<>,.?/~`|\\\\]+$"
    minLength: 32
    requireUpperCase: true
    requireSpecialChar: true
```

**Step 9: Deploy Fix**

```bash
# Update code to handle new password format
git commit -m "fix: support new password format from rotation"
git push origin main

# CI/CD automatically deploys
# After 5 minutes: Version 4 deployed
# Next rotation (30 min): Should succeed
# Monitor next rotation...
```

**Step 10: Prevent Future Issues**

```bash
# Add pre-deployment tests
# helm/values-prod.yaml
preDeploymentTests:
  - name: database-connectivity
    command: psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT 1"
  - name: password-validation
    command: validate-password-format $DB_PASSWORD

# Add alerting for rotation failures
# CloudWatch alarm: SecretsManager rotation failed

# Add monitoring for password validation errors
# Application metric: auth.password_validation_errors

# Add chaos engineering test monthly
# Simulate credential rotation failure, verify app handles gracefully
```

---

## Key Takeaways

### Integration Points

1. **Code Push** → GitHub
2. **GitHub Actions** → Build, test, scan, push image
3. **ECR** → Store container images
4. **Terraform** → Define infrastructure
5. **Helm** → Deploy applications
6. **Kubernetes** → Run workloads
7. **Istio** → Manage traffic, implement canary
8. **OpenTelemetry** → Collect metrics and traces
9. **Prometheus/Grafana** → Visualize metrics
10. **CloudWatch** → AWS-level monitoring
11. **Alerts** → Trigger incident response

### Failure Points and Mitigations

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Pod crash | Readiness probe fails | Kubelet restarts pod |
| Database unavailable | Health check timeout | Istio outlier detection |
| Node failure | Heartbeat timeout | Pods rescheduled to other nodes |
| Bad deployment | Error rate spike | Automatic canary rollback or helm rollback |
| Credential rotation failure | Auth errors | Helm rollback to previous version |

### Operational Best Practices

- **Always test deployments** before production (DEV → QA → PROD)
- **Use gradual rollouts** (canary) for critical services
- **Monitor everything** (metrics, logs, traces)
- **Have runbooks** for common incidents
- **Do chaos engineering** to find issues before they hit prod
- **Automate remediation** where safe (scaling, rollbacks)

