# Kubernetes Monitoring, Scanning, and Cloud Tools Guide

## Table of Contents
1. [Open Source Monitoring & Observability](#open-source-monitoring--observability)
2. [Service Mesh & Advanced Networking](#service-mesh--advanced-networking)
3. [Security Scanning & Runtime Protection](#security-scanning--runtime-protection)
4. [AWS Cloud-Native Tools](#aws-cloud-native-tools)
5. [Azure Cloud-Native Tools](#azure-cloud-native-tools)
6. [HashiCorp Vault & Secrets Management](#hashicorp-vault--secrets-management)
7. [Integration Patterns](#integration-patterns)
8. [Interview Questions](#interview-questions)
9. [Appendix: Plan-required deep dives](#appendix-plan-required-deep-dives-eksaks-interview-depth)

---

## Part 1: Open Source Monitoring & Observability

### 1.1 Prometheus - Metrics Collection

**What it is:** Time-series database and scraper for collecting Kubernetes metrics.

**Architecture:**
```
┌─────────────┐
│  Prometheus │  ◄── Scrapes every 15s
├─────────────┤
│ Time-Series │  ◄── Stores metrics with timestamp
│  Database   │
└─────────────┘
      ▲
      │ (scrape targets)
      │
   ┌──┴───┬──────────┬──────────┐
   │      │          │          │
 Kubelet Pod   Service  Node
```

**Installation (Helm):**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values prometheus-values.yaml
```

**Helm Values Example:**
```yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

grafana:
  enabled: true
  adminPassword: "your-secure-password"
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-operated:9090
        isDefault: true
```

**Query Examples (PromQL):**
```promql
# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod_name)

# Memory usage
sum(container_memory_usage_bytes) by (pod_name) / 1024 / 1024

# Pod restart count
increase(kube_pod_container_status_restarts_total[15m]) > 0

# Requests per second
sum(rate(http_requests_total[1m])) by (service)
```

**ServiceMonitor for Custom Scraping:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
  namespace: default
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

**Interview Questions:**
- Q: How does Prometheus decide when to scrape a target?
  A: Via the `scrape_configs` section or ServiceMonitor CRD. It scrapes at intervals (default 15s) and expects targets to expose metrics at `/metrics` endpoint.

- Q: What's the difference between `rate()` and `increase()`?
  A: `rate()` returns average increase per second over time window. `increase()` returns raw increase over the time window.

---

### 1.2 Grafana - Visualization & Dashboards

**What it is:** Dashboard and visualization platform for metrics.

**Key Dashboards for Kubernetes:**
- `kube-system-cluster-monitoring` - Cluster overview
- `pod-metrics` - Individual pod metrics
- `node-exporter` - Node-level metrics
- `prometheus` - Prometheus health

**Provisioning Dashboards via ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  pod-metrics.json: |
    {
      "dashboard": {
        "title": "Pod Metrics",
        "panels": [
          {
            "title": "CPU Usage",
            "targets": [
              {
                "expr": "sum(rate(container_cpu_usage_seconds_total[5m])) by (pod_name)"
              }
            ]
          }
        ]
      }
    }
```

**Alert Rules:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-alerts
spec:
  groups:
  - name: pod.rules
    interval: 30s
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"

    - alert: HighCPUUsage
      expr: sum(rate(container_cpu_usage_seconds_total[5m])) by (pod) > 0.9
      for: 10m
      annotations:
        summary: "Pod {{ $labels.pod }} CPU usage is > 90%"
```

---

### 1.3 Loki - Log Aggregation

**What it is:** Horizontally-scalable log aggregation system optimized for Kubernetes.

**Architecture:**
```
┌─────────────────┐
│  Fluent Bit     │  ◄── Collects logs from all pods
│  (DaemonSet)    │
└────────┬────────┘
         │
    ┌────▼─────┐
    │   Loki   │  ◄── Stores logs with label-based indexing
    ├──────────┤
    │ BlockDB  │
    │ (S3)     │
    └────┬─────┘
         │
    ┌────▼──────────┐
    │ Grafana Loki  │
    │ Plugin        │
    └───────────────┘
```

**Installation:**
```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --values loki-values.yaml
```

**Loki Values:**
```yaml
loki:
  enabled: true
  persistence:
    enabled: true
    size: 100Gi
  config:
    auth_enabled: false
    ingester:
      chunk_idle_period: 3m
      max_chunk_age: 1h
    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h

promtail:
  enabled: true
  config:
    clients:
    - url: http://loki:3100/loki/api/v1/push
    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

**LogQL Query Examples:**
```logql
# All logs from nginx pod
{pod="nginx"}

# Error logs
{pod="nginx"} |= "error"

# Response time > 1s
{job="api"} |= "duration" | duration > 1000

# Count errors per namespace
count_over_time({severity="error"} [5m]) by (namespace)

# Rate of errors
rate({level="error"}[5m])
```

---

### 1.4 ELK Stack - Elasticsearch, Logstash, Kibana

**When to use:** Larger deployments needing advanced log analytics.

**Deployment Architecture:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: logging
data:
  logstash.conf: |
    input {
      beats {
        port => 5044
      }
    }
    
    filter {
      if [type] == "kube-logs" {
        mutate {
          add_field => { "[@metadata][index_name]" => "k8s-logs-%{+YYYY.MM.dd}" }
        }
      }
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch:9200"]
        index => "%{[@metadata][index_name]}"
        user => "elastic"
        password => "${ES_PASSWORD}"
      }
    }
```

**Elasticsearch Pod:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
        env:
        - name: discovery.type
          value: "zen"
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: xpack.security.enabled
          value: "true"
        - name: ELASTIC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-secret
              key: password
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp2
      resources:
        requests:
          storage: 50Gi
```

---

### 1.5 Jaeger - Distributed Tracing

**What it is:** Distributed tracing system for microservices.

**Installation:**
```bash
kubectl apply -f https://github.com/jaegertracing/jaeger-kubernetes/releases/download/v1.30.0/jaeger-operator-bundle.yaml

kubectl create ns observability

cat <<EOF | kubectl apply -f -
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: info
  storage:
    type: elasticsearch
    elasticsearch:
      nodePort: 9200
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
    - jaeger.example.com
EOF
```

**Instrumenting Application (Python):**
```python
from jaeger_client import Config

def init_tracer(service_name):
    config = Config(
        config={
            'sampler': {
                'type': 'const',
                'param': 1,
            },
            'logging': True,
            'local_agent': {
                'reporting_host': 'jaeger-agent',
                'reporting_port': 6831,
            },
        },
        service_name=service_name,
        validate=True,
    )
    return config.initialize_tracer()

tracer = init_tracer('my-service')

with tracer.start_active_span('my-operation') as scope:
    span = scope.span
    span.set_tag('http.method', 'GET')
    span.set_tag('http.url', '/api/users')
    # Do work
    span.log_kv({'event': 'request_processed'})
```

---

## Part 2: Service Mesh & Advanced Networking

### 2.1 Istio - Service Mesh

**What it is:** Service mesh providing traffic management, security, observability.

**Architecture:**
```
Pod1 (with Envoy sidecar)
    ├─ Container: app
    └─ Container: envoy-proxy (intercepts all traffic)
         │
         ├─ Traffic Management (VirtualService)
         ├─ Security (PeerAuthentication)
         ├─ Observability (metrics to Prometheus)
         └─ Load Balancing
```

**Installation:**
```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Install Istio
istioctl install --set profile=demo -y

# Label namespace for sidecar injection
kubectl label namespace default istio-injection=enabled
```

**VirtualService for Canary Deployment:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: myapp
        port:
          number: 8080
        subset: v1
      weight: 90
    - destination:
        host: myapp
        port:
          number: 8080
        subset: v2
      weight: 10

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Mutual TLS (mTLS):**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-get
  namespace: default
spec:
  selector:
    matchLabels:
      app: myapp
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/frontend"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/*"]
```

---

### 2.2 Cilium - eBPF-Based Networking

**What it is:** eBPF-based CNI providing networking, security, and observability.

**Installation:**
```bash
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium \
  --namespace kube-system \
  --values cilium-values.yaml
```

**Cilium Values:**
```yaml
cilium:
  image:
    tag: "1.13.0"
  
  daemon:
    runPath: /var/run/cilium
  
  # Enable Hubble for observability
  hubble:
    enabled: true
    relay:
      enabled: true
    ui:
      enabled: true
  
  # Enable network policies
  networkPolicy: "cilium"
  
  # Enable service mesh features
  l7Proxy: true
  
  # Performance tuning
  bpf:
    progs:
      limit: 2048
```

**Cilium Network Policy:**
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-access
  namespace: default
spec:
  description: "Allow API access only from frontend"
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/.*"
  egress:
  - toEndpoints:
    - matchLabels:
        app: database
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP
```

**Hubble Observability:**
```bash
# View live traffic
cilium hubble observe --follow

# Show dropped packets
cilium hubble observe --verdict=DROPPED

# Filter by namespace
cilium hubble observe --namespace=default
```

---

## Part 3: Security Scanning & Runtime Protection

### 3.1 Trivy - Image and Filesystem Scanning

**What it is:** Comprehensive vulnerability scanner for images, filesystems, and repositories.

**Installation:**
```bash
# Install Trivy CLI
wget https://github.com/aquasecurity/trivy/releases/download/v0.40.0/trivy_0.40.0_Linux-64bit.tar.gz
tar zxvf trivy_0.40.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/
```

**Scan Container Image:**
```bash
# Scan image for vulnerabilities
trivy image nginx:latest

# Generate JSON report
trivy image --format json --output report.json nginx:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan private registry
trivy image --registry-username user --registry-password pass \
  private-registry/myimage:v1.0
```

**Integration in CI/CD (GitHub Actions):**
```yaml
name: Trivy Scan
on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - name: Run Trivy scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myregistry/myimage:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
```

**Trivy Admission Controller in Kubernetes:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: trivy-scan-job
spec:
  template:
    spec:
      containers:
      - name: trivy
        image: aquasec/trivy:latest
        args:
        - image
        - --format
        - json
        - --exit-code
        - "0"  # 0 = always exit success, 1 = fail on vulnerabilities
        - --severity
        - "HIGH,CRITICAL"
        - --image-ref
        - myimage:latest
      restartPolicy: Never
  backoffLimit: 3
```

---

### 3.2 Snyk - Container and Code Scanning

**What it is:** Developer-first security platform for scanning code, dependencies, and containers.

**Installation:**
```bash
# Install Snyk CLI
npm install -g snyk

# Authenticate
snyk auth

# Scan Docker image
snyk container test myimage:latest

# Scan Kubernetes manifests
snyk iac test deployment.yaml

# Generate report
snyk container test myimage:latest --json > snyk-report.json
```

**Snyk in CI/CD:**
```yaml
# .github/workflows/snyk-scan.yml
name: Snyk Security Scan
on: [push, pull_request]

jobs:
  snyk:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run Snyk test
      uses: snyk/actions/docker@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        image: myregistry/myimage:latest
        args: --severity=high
    
    - name: Upload results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'snyk.sarif'
```

---

### 3.3 Falco - Runtime Security

**What it is:** Runtime security and anomaly detection for Kubernetes.

**Installation:**
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --values falco-values.yaml
```

**Falco Values:**
```yaml
falco:
  image:
    tag: "0.35.1"
  
  # Enable eBPF probe
  ebpf:
    enabled: true
  
  # Custom rules
  rulesFile: /etc/falco/rules.yaml
  
  # Outputs
  outputs:
    - stdout

  # JSON output
  jsonOutput: true

falcoExporter:
  enabled: true
  port: 5668

# Send alerts to external systems
falco:
  grpc:
    enabled: true
  grpcOutput:
    enabled: true
```

**Custom Falco Rules:**
```yaml
- rule: Unauthorized Process Execution
  desc: Detect unauthorized process execution
  condition: spawned_process and container and proc.name not in (allowed_processes)
  output: >
    Unauthorized process started
    (user=%user.name command=%proc.cmdline container=%container.info)
  priority: WARNING

- rule: Suspicious File Access
  desc: Detect suspicious file access
  condition: open and container and fd.name glob "/etc/shadow*"
  output: >
    Sensitive file accessed
    (user=%user.name file=%fd.name container=%container.info)
  priority: CRITICAL
```

---

## Part 4: AWS Cloud-Native Tools

### 4.1 AWS Secrets Manager

**What it is:** Managed secrets storage for AWS resources.

**Store a Secret:**
```bash
aws secretsmanager create-secret \
  --name prod/db/password \
  --secret-string '{"username":"admin","password":"MyP@ssw0rd"}' \
  --region us-east-1
```

**Retrieve Secret in Application:**
```python
import boto3
import json

client = boto3.client('secretsmanager', region_name='us-east-1')

def get_secret(secret_name):
    try:
        response = client.get_secret_value(SecretId=secret_name)
        if 'SecretString' in response:
            return json.loads(response['SecretString'])
    except Exception as e:
        print(f"Error retrieving secret: {e}")
        return None

# Usage
db_creds = get_secret('prod/db/password')
print(f"Username: {db_creds['username']}")
```

**IRSA Pod Access:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/app-role

---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: AWS_ROLE_ARN
      value: "arn:aws:iam::ACCOUNT:role/app-role"
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: "/var/run/secrets/eks.amazonaws.com/serviceaccount/token"
    volumeMounts:
    - name: aws-token
      mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
  volumes:
  - name: aws-token
    projected:
      sources:
      - serviceAccountToken:
          audience: sts.amazonaws.com
          expirationSeconds: 86400
          path: token
```

**IAM Policy for Secret Access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:prod/*"
    }
  ]
}
```

---

### 4.2 AWS Systems Manager Parameter Store

**What it is:** Centralized configuration and secrets management.

**Store Parameter:**
```bash
# Standard parameter (4KB max)
aws ssm put-parameter \
  --name /app/config/db-host \
  --value "db.example.com" \
  --type "String"

# Secure parameter (encrypted)
aws ssm put-parameter \
  --name /app/secrets/api-key \
  --value "sk_live_abcd1234" \
  --type "SecureString" \
  --key-id alias/aws/ssm
```

**Retrieve Parameter:**
```bash
aws ssm get-parameter \
  --name /app/config/db-host \
  --query 'Parameter.Value' \
  --output text

# Get multiple parameters
aws ssm get-parameters-by-path \
  --path "/app/config" \
  --recursive
```

---

### 4.3 AWS KMS (Key Management Service)

**What it is:** Key management for encryption/decryption.

**Create Encryption Key:**
```bash
aws kms create-key \
  --description "EKS Secrets Encryption" \
  --origin AWS_KMS

# Create alias
aws kms create-alias \
  --alias-name alias/eks-secrets \
  --target-key-id abc12345-1234-1234-1234-123456789012
```

**Enable KMS Encryption in EKS:**
```bash
aws eks update-cluster-config \
  --name my-cluster \
  --encryption-config resources=secrets,provider={keyArn=arn:aws:kms:us-east-1:ACCOUNT:key/KEY_ID}
```

**Encrypt/Decrypt Data:**
```python
import boto3
import base64

kms_client = boto3.client('kms', region_name='us-east-1')

# Encrypt
response = kms_client.encrypt(
    KeyId='alias/eks-secrets',
    Plaintext=b'sensitive-data'
)
encrypted_data = base64.b64encode(response['CiphertextBlob']).decode()

# Decrypt
decrypt_response = kms_client.decrypt(
    CiphertextBlob=base64.b64decode(encrypted_data)
)
plaintext = decrypt_response['Plaintext'].decode()
```

---

### 4.4 AWS CloudWatch - Monitoring & Logging

**What it is:** AWS native monitoring, logging, and alerting.

**CloudWatch Agent ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudwatch-config
  namespace: amazon-cloudwatch
data:
  cwagentconfig.json: |
    {
      "agent": {
        "region": "us-east-1",
        "debug": false
      },
      "metrics": {
        "namespace": "EKS/Cluster",
        "metrics_collected": {
          "cpu": {
            "measurement": [
              {
                "name": "cpu_usage_idle",
                "rename": "CPU_IDLE",
                "unit": "Percent"
              }
            ],
            "metrics_collection_interval": 60
          },
          "mem": {
            "measurement": [
              {
                "name": "mem_used_percent",
                "rename": "MEM_USED",
                "unit": "Percent"
              }
            ],
            "metrics_collection_interval": 60
          }
        }
      },
      "logs": {
        "logs_collected": {
          "files": {
            "collect_list": [
              {
                "file_path": "/var/log/containers/*.log",
                "log_group_name": "/aws/eks/cluster",
                "log_stream_name": "{instance_id}"
              }
            ]
          }
        }
      }
    }
```

**CloudWatch Logs Insights Queries:**
```
# Find errors
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() as error_count by bin(5m)

# Latency analysis
fields @duration
| stats avg(@duration), max(@duration), pct(@duration, 99) as p99

# Pod restart events
fields @timestamp, kubernetes.pod_name
| filter @message like /restart/
| stats count() as restart_count by kubernetes.pod_name
```

**CloudWatch Alarms:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-pod-memory \
  --alarm-description "Alert when pod memory > 80%" \
  --metric-name memory_utilization \
  --namespace EKS/Cluster \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:alerts
```

---

### 4.5 AWS X-Ray - Distributed Tracing

**What it is:** AWS-native distributed tracing service.

**X-Ray Daemon DaemonSet:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: xray-daemon
  namespace: amazon-cloudwatch
spec:
  selector:
    matchLabels:
      app: xray-daemon
  template:
    metadata:
      labels:
        app: xray-daemon
    spec:
      containers:
      - name: xray-daemon
        image: public.ecr.aws/xray/aws-xray-daemon:latest
        ports:
        - name: udp
          containerPort: 2000
          protocol: UDP
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
```

**Instrument Application (Python):**
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()

@xray_recorder.capture('process_request')
def process_request(request_data):
    # X-Ray will automatically capture this
    segment = xray_recorder.current_segment()
    segment.put_annotation('user_id', request_data['user_id'])
    segment.put_metadata('request_body', request_data)
    
    # Your logic here
    return process_data(request_data)
```

---

### 4.6 AWS ECR - Container Registry

**What it is:** AWS managed container registry.

**Push Image to ECR:**
```bash
# Create repository
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1

# Get login token
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  ACCOUNT.dkr.ecr.us-east-1.amazonaws.com

# Tag and push image
docker tag myapp:latest ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
docker push ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0

# Enable image scanning
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1
```

---

### 4.7 AWS GuardDuty - Threat Detection

**What it is:** Intelligent threat detection and response.

**Enable GuardDuty:**
```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --region us-east-1

# List findings
aws guardduty list-findings \
  --detector-id <detector-id> \
  --finding-criteria '{"Criterion":{"severity":{"Gte":5}}}' \
  --region us-east-1
```

**GuardDuty for EKS:**
```bash
# Enable EKS protection
aws guardduty create-detector \
  --enable \
  --findings-export-options '{"S3Destination":{"Bucket":"my-bucket","KeyPrefix":"guardduty"}}' \
  --eks-addon-management '{"EksClusterName":"my-cluster","EksAnomalyDetection":"ENABLED"}' \
  --region us-east-1
```

---

### 4.8 AWS Security Hub & Config

**Security Hub:**
```bash
# Enable Security Hub
aws securityhub enable-security-hub --region us-east-1

# Get compliance summary
aws securityhub get-compliance-summary --region us-east-1

# Get findings
aws securityhub get-findings \
  --filters '{"ComplianceStatus":[{"Value":"FAILED"}]}' \
  --region us-east-1
```

**AWS Config:**
```bash
# Enable Config
aws configservice put-config-recorder \
  --config-recorder name=default,roleARN=arn:aws:iam::ACCOUNT:role/ConfigRole,recordingGroup={allSupported=true}

# Create Config rules
aws configservice put-config-rule \
  --config-rule file://rule.json
```

---

## Part 5: Azure Cloud-Native Tools

### 5.1 Azure Key Vault

**What it is:** Secure key management and secrets storage.

**Create Key Vault:**
```bash
az keyvault create \
  --name mykeyvault \
  --resource-group mygroup \
  --location eastus \
  --sku standard
```

**Store Secret:**
```bash
az keyvault secret set \
  --vault-name mykeyvault \
  --name db-password \
  --value 'MySecureP@ssw0rd'

# Retrieve secret
az keyvault secret show \
  --vault-name mykeyvault \
  --name db-password \
  --query value -o tsv
```

**Azure Key Vault Provider for Secrets Store CSI Driver:**
```yaml
apiVersion: v1
kind: SecretProviderClass
metadata:
  name: azure-kvs
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: mykeyvault
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectAlias: DB_PASSWORD
        - |
          objectName: api-key
          objectType: secret
          objectAlias: API_KEY
    tenantId: "TENANT_ID"

---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: workload-identity-sa
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets-store
      mountPath: /mnt/secrets-store
      readOnly: true
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: secrets-store
          key: DB_PASSWORD
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-kvs"
```

---

### 5.2 Azure Application Insights

**What it is:** Application performance monitoring and logging.

**Instrument Application (Python):**
```python
from opencensus.ext.azure.log_exporter import AzureLogHandler
from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer

import logging

# Configure logging
handler = AzureLogHandler(connection_string="InstrumentationKey=...")
logger = logging.getLogger(__name__)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Configure tracing
tracer = Tracer(
    exporter=AzureExporter(connection_string="InstrumentationKey=..."),
    sampler=ProbabilitySampler(1.0)
)

# Use tracer
with tracer.span(name="process_request") as span:
    span.add_attribute("user_id", "12345")
    logger.info("Processing request")
```

---

### 5.3 Azure Log Analytics

**What it is:** Log aggregation and analysis.

**Create Log Analytics Workspace:**
```bash
az monitor log-analytics workspace create \
  --resource-group mygroup \
  --workspace-name myworkspace \
  --location eastus
```

**KQL Query Examples:**
```kusto
# Pod CPU usage
ContainerLog
| where PodName contains "myapp"
| summarize AvgCpu = avg(CpuPercent) by PodName, bin(TimeGenerated, 5m)

# Error count by namespace
ContainerLog
| where LogLevel == "ERROR"
| summarize ErrorCount = count() by Namespace, bin(TimeGenerated, 10m)

# Performance metrics
Perf
| where ObjectName == "K8SNode"
| summarize AvgCPU = avg(CounterValue) by Computer
```

---

### 5.4 Azure Container Registry

**What it is:** Managed container registry for Azure.

**Create Registry:**
```bash
az acr create \
  --resource-group mygroup \
  --name myregistry \
  --sku Standard
```

**Build and Push Image:**
```bash
# Build image using ACR
az acr build \
  --registry myregistry \
  --image myapp:v1.0 \
  .

# Push local image
docker tag myapp:latest myregistry.azurecr.io/myapp:v1.0
az acr login --name myregistry
docker push myregistry.azurecr.io/myapp:v1.0
```

---

### 5.5 Microsoft Defender for Cloud

**What it is:** Cloud security posture management and threat protection.

**Enable Defender for Containers:**
```bash
az security auto-provisioning-setting update \
  --auto-provision "On" \
  --auto-provision-setting-name default
```

**Get Recommendations:**
```bash
az security assessment list \
  --query "[].{Name:name, Status:status.code}" -o table
```

---

### 5.6 Azure Policies & Policy as Code

**What it is:** Enforce organizational standards.

**Define Policy:**
```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "field": "type",
      "equals": "Microsoft.Compute/virtualMachines"
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

---

## Part 6: HashiCorp Vault & Secrets Management

### 6.1 HashiCorp Vault Installation

**What it is:** Universal secrets management platform.

**Install Vault in Kubernetes:**
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --values vault-values.yaml
```

**Vault Values:**
```yaml
server:
  dataStorage:
    size: 10Gi
  
  ha:
    enabled: true
    replicas: 3
  
  ui:
    enabled: true
    serviceType: LoadBalancer

ui:
  enabled: true

injector:
  enabled: true
```

**Initialize and Unseal:**
```bash
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3

# Unseal
kubectl exec -n vault vault-0 -- vault operator unseal <KEY1>
kubectl exec -n vault vault-0 -- vault operator unseal <KEY2>
kubectl exec -n vault vault-0 -- vault operator unseal <KEY3>
```

### 6.2 Vault Kubernetes Authentication

**Configure Kubernetes Auth Method:**
```bash
# Enable auth method
vault auth enable kubernetes

# Configure
vault write auth/kubernetes/config \
  token_reviewer_jwt="@/var/run/secrets/kubernetes.io/serviceaccount/token" \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create role
vault write auth/kubernetes/role/app-role \
  bound_service_account_names=app \
  bound_service_account_namespaces=default \
  policies=app-policy \
  ttl=24h
```

**Pod with Vault Injection:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: default

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: default
spec:
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/database/config"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/database/config" -}}
          export DB_HOST="{{ .Data.data.host }}"
          export DB_USER="{{ .Data.data.username }}"
          export DB_PASS="{{ .Data.data.password }}"
          {{- end }}
        vault.hashicorp.com/role: "app-role"
    spec:
      serviceAccountName: app
      containers:
      - name: app
        image: myapp:latest
        command:
        - /bin/sh
        - -c
        - |
          source /vault/secrets/database
          exec /app/start.sh
```

---

## Part 7: Integration Patterns

### Complete Observability Stack

```yaml
# Deploy all monitoring components
apiVersion: v1
kind: Namespace
metadata:
  name: observability

---
# Prometheus
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: observability
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes'
      kubernetes_sd_configs:
      - role: pod

---
# Loki
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: loki-pvc
  namespace: observability
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi

---
# Jaeger
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: observability
spec:
  ports:
  - port: 6831
    protocol: UDP
  selector:
    app: jaeger
```

### Complete Security Stack

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: security

---
# Falco
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
  namespace: security
data:
  custom-rules.yaml: |
    - rule: Unauthorized Process
      condition: spawned_process and container
      output: Unauthorized process (user=%user.name)
      priority: WARNING

---
# Trivy scanning job
apiVersion: batch/v1
kind: CronJob
metadata:
  name: trivy-scan
  namespace: security
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: trivy
            image: aquasec/trivy:latest
            args:
            - image
            - --severity
            - "HIGH,CRITICAL"
            - myregistry/myapp:latest
          restartPolicy: OnFailure
```

---

## Part 8: Interview Questions

### Monitoring Tools Questions

**Q1: Explain the difference between Prometheus and Grafana. When would you use each?**
A: 
- **Prometheus**: Time-series database and metric scraper. Used for data collection, storage, and querying. Provides PromQL for complex queries.
- **Grafana**: Visualization platform. Used to create dashboards, alerts, and visualizations of metrics. Can use Prometheus as data source.
- Together: Prometheus collects and stores metrics, Grafana visualizes them.

**Q2: What are the advantages of Loki over ELK stack?**
A:
- Label-based indexing (more efficient than full-text)
- Lower resource consumption
- Better suited for cloud-native/container environments
- Simpler deployment in Kubernetes
- ELK is better for complex log analytics and full-text search

**Q3: How would you implement distributed tracing across microservices?**
A:
- Instrument each service with tracing library (Jaeger SDK, OpenTelemetry)
- Configure trace exporter to send to Jaeger backend
- Pass trace context (trace-id, span-id) between services
- Query traces in Jaeger UI to debug latency issues

### Service Mesh Questions

**Q4: What problems does Istio solve in Kubernetes?**
A:
- Traffic management (canary deployments, A/B testing)
- Security (mTLS between services)
- Observability (metrics, traces, logs)
- Resilience (retries, timeouts, circuit breakers)
- Service discovery and load balancing

**Q5: Explain the difference between Istio and Cilium.**
A:
- **Istio**: Service mesh at application layer, uses Envoy sidecars for traffic control
- **Cilium**: CNI and eBPF-based security at network layer
- Istio: More features but higher overhead
- Cilium: Lighter weight, better performance, network-level security

### Security Scanning Questions

**Q6: How would you integrate vulnerability scanning in your CI/CD pipeline?**
A:
- Use Trivy/Snyk in GitHub Actions or Jenkins
- Scan before pushing image to registry
- Fail build if HIGH/CRITICAL vulnerabilities found
- Generate SARIF reports for GitHub Security tab
- Implement exception process for acceptable risks

**Q7: What's the difference between Trivy and Snyk?**
A:
- **Trivy**: Free, comprehensive, good for images and filesystems
- **Snyk**: Paid (free tier available), integrates with development workflow, includes code scanning
- Trivy: Better for CI/CD automation
- Snyk: Better for developer experience

### Cloud Tools Questions

**Q8: When would you use AWS Secrets Manager vs Parameter Store?**
A:
- **Secrets Manager**: 
  - Automatic rotation
  - Encryption required
  - Secrets databases, API keys, credentials
  - Higher cost
- **Parameter Store**:
  - Simple configuration management
  - No automatic rotation
  - Can be unencrypted (Standard) or encrypted (SecureString)
  - Lower cost, good for non-sensitive config

**Q9: Explain IRSA and its benefits over IAM user credentials.**
A:
- IRSA: IAM Roles for Service Accounts
- Benefits:
  - No credentials in pod specs or ConfigMaps
  - Each pod can have unique IAM permissions
  - Automatic credential rotation via OIDC
  - Better security posture
  - Fine-grained access control

**Q10: How would you implement secrets management in Kubernetes using multiple providers?**
A:
- Use External Secrets Operator
- Configure providers (AWS Secrets Manager, Azure Key Vault, Vault)
- Create SecretStore per provider
- Define ExternalSecret to sync secrets
- Supports secret rotation and templating

---

## Appendix: Plan-required deep dives (EKS/AKS interview depth)

This section maps explicitly to the **Kubernetes Interview Preparation Plan** items: AlertManager, Elasticsearch/Kibana positioning, OpenTelemetry, OPA/Gatekeeper, kube-bench, and cross-links service mesh / cloud sections above.

### Alertmanager (with Prometheus / kube-prometheus-stack)

**Role:** Receives alerts from Prometheus, **deduplicates**, **routes** to receivers (PagerDuty, Slack, email), supports **silences** and **inhibition**.

**Typical Helm (kube-prometheus-stack):** Alertmanager is deployed as a StatefulSet; Prometheus `alertmanager.config` or `alertmanagerFiles` supplies YAML.

**Minimal config concepts:**
- **route:** `receiver`, `match` / `match_re`, `continue`, `routes` (sub-routes)
- **receiver:** `slack_configs`, `email_configs`, `pagerduty_configs`
- **inhibit_rules:** suppress warnings when critical fires

```yaml
global:
  resolve_timeout: 5m
route:
  receiver: "default"
  group_by: ["alertname", "cluster", "namespace"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  routes:
  - match:
      severity: critical
    receiver: pagerduty
receivers:
- name: default
  slack_configs:
  - api_url: "${SLACK_WEBHOOK}"
    channel: "#alerts"
```

**Interview:** Difference between Prometheus alerting rules and Alertmanager? Prometheus *evaluates* rules and *fires* alerts; Alertmanager *routes*, *groups*, *deduplicates*, and *notifies*.

---

### OpenTelemetry (OTel) — tracing stack (with Jaeger)

**Architecture:**
- **Instrumentation:** App uses OTel API/SDK (language-specific) → produces traces/metrics/logs.
- **Collector:** `opentelemetry-collector` receives OTLP (gRPC/HTTP), processes (batch, filter), **exports** to Jaeger, Prometheus, Tempo, cloud backends.

**Why OTel:** Vendor-neutral; **W3C Trace Context** (`traceparent` header) for correlation across services and meshes.

**Collector pipeline snippet (conceptual):**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:
exporters:
  otlp:
    endpoint: jaeger-collector.observability.svc:4317
    tls:
      insecure: true
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp]
```

**Interview:** “Jaeger vs OTel?” — Jaeger is a **backend/UI**; OTel is **instrumentation + collector standard**. Deploy Jaeger with **OTLP** ingest to accept OTel exports natively (Jaeger v1.35+).

---

### OPA Gatekeeper — policy as code (Kubernetes)

**Components:**
- **ConstraintTemplate** (Rego): defines the policy schema and logic.
- **Constraint**: binds template to namespaces / kinds (enforces).

**Example:** Require `runAsNonRoot` in Pod security context:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirednonroot
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredNonRoot
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirednonroot
        violation[{"msg": msg}] {
          not input.review.object.spec.securityContext.runAsNonRoot
          msg := "Pods must set spec.securityContext.runAsNonRoot: true"
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredNonRoot
metadata:
  name: require-non-root
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

**Versus OPA standalone:** Gatekeeper = OPA + **K8s CRDs** + **audit** of existing resources.

---

### kube-bench — CIS Kubernetes benchmark

**Purpose:** Run **CIS checks** against API server, etcd, nodes (kubelet), etc.

**Typical usage:**
```bash
# Job or DaemonSet runs aquasec/kube-bench against CIS profile
kube-bench run --targets master,node,etcd,policies
```

**Output:** PASS/WARN/FAIL per check ID (maps to CIS PDF). **Interview:** Used for **compliance evidence** and hardening gaps—not runtime threat detection (that’s Falco).

---

## Summary Table

| Tool | Purpose | When to Use | Overhead |
|------|---------|------------|----------|
| Prometheus | Metrics collection | Always for monitoring | Medium |
| Grafana | Visualization | Always for dashboards | Low |
| Alertmanager | Alert routing & grouping | With Prometheus | Low |
| Loki | Log aggregation | Cloud-native deployments | Low |
| Elasticsearch/Kibana | Log search & analytics | Large-scale ELK | High |
| Jaeger | Distributed tracing (backend/UI) | Microservices debugging | Medium |
| OpenTelemetry | Instrumentation & collector | Standardized traces/metrics/logs | Low–Medium |
| Istio | Service mesh | Advanced traffic management | High |
| Cilium | Network policy/security | High-security environments | Low–Medium |
| Trivy | Image scanning | CI/CD pipeline | Very Low |
| Falco | Runtime security | Threat detection | Medium |
| OPA Gatekeeper | Admission policies | Org-wide K8s policy | Medium |
| kube-bench | CIS compliance checks | Audits / hardening | Low |
| AWS Secrets Manager | Secrets storage | AWS deployments | None (managed) |
| Azure Key Vault | Secrets storage | Azure deployments | None (managed) |
| Vault | Universal secrets | Multi-cloud | Medium |

---

**Study Time Estimate:** 8-10 hours
**Key Takeaway:** Most organizations use multiple tools together - metrics (Prometheus), visualization (Grafana), logs (Loki), and traces (Jaeger) form a complete observability stack.
