# 06 - Hands-On Labs: Practical EKS Exercises

## Lab Prerequisites

Before starting, ensure you have:
- Access to an EKS cluster
- kubectl configured and working
- helm installed (3.x)
- istioctl installed
- git for cloning examples

```bash
# Verify setup
kubectl cluster-info
kubectl get nodes
helm version
istioctl version
```

---

## Lab 1: Deploy Multi-Environment Setup with Helm

**Duration:** 20-30 minutes
**Goal:** Deploy a microservice to DEV, QA, and Production using Helm

### Step 1: Create Namespaces

```bash
kubectl create namespace dev
kubectl create namespace qa
kubectl create namespace production
kubectl label namespace production environment=prod
kubectl label namespace qa environment=qa
kubectl label namespace dev environment=dev
```

### Step 2: Create Helm Chart Structure

```bash
mkdir -p payment-service-chart/{templates,charts}
cd payment-service-chart

cat > Chart.yaml <<EOF
apiVersion: v2
name: payment-service
description: Payment microservice
type: application
version: 1.0.0
appVersion: "1.0.0"
EOF

cat > values.yaml <<EOF
replicaCount: 1
image:
  repository: nginx
  tag: "latest"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
EOF

cat > values-dev.yaml <<EOF
replicaCount: 1
EOF

cat > values-qa.yaml <<EOF
replicaCount: 2
EOF

cat > values-prod.yaml <<EOF
replicaCount: 3
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
EOF
```

### Step 3: Create Deployment Template

```bash
cat > templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "payment-service.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "payment-service.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "payment-service.name" . }}
        version: v1
    spec:
      containers:
      - name: payment-service
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
EOF

cat > templates/_helpers.tpl <<'EOF'
{{- define "payment-service.name" -}}
payment-service
{{- end }}

{{- define "payment-service.fullname" -}}
payment-service
{{- end }}
EOF
```

### Step 4: Create Service Template

```bash
cat > templates/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: {{ include "payment-service.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
  selector:
    app: {{ include "payment-service.name" . }}
EOF
```

### Step 5: Deploy to All Environments

```bash
# DEV environment
helm install payment-service . \
  -n dev \
  -f values.yaml \
  -f values-dev.yaml

# QA environment
helm install payment-service . \
  -n qa \
  -f values.yaml \
  -f values-qa.yaml

# Production environment
helm install payment-service . \
  -n production \
  -f values.yaml \
  -f values-prod.yaml
```

### Step 6: Verify Deployments

```bash
# Check all environments
kubectl get deployments -n dev
kubectl get deployments -n qa
kubectl get deployments -n production

# Expected output:
# NAME                 READY   UP-TO-DATE   AVAILABLE
# payment-service      1/1     1            1         (DEV - 1 replica)
# payment-service      2/2     2            2         (QA - 2 replicas)
# payment-service      3/3     3            3         (PROD - 3 replicas)

# View resources
kubectl describe deployment -n production payment-service
```

### Step 7: Cleanup

```bash
helm uninstall payment-service -n dev
helm uninstall payment-service -n qa
helm uninstall payment-service -n production
```

**✓ Lab Complete:** You've deployed the same application to 3 environments with different configurations!

---

## Lab 2: Install and Configure Istio Service Mesh

**Duration:** 15-20 minutes
**Goal:** Set up Istio and enable sidecar injection

### Step 1: Download Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
istioctl version
```

### Step 2: Install Istio Control Plane

```bash
# Install Istio in production configuration
istioctl install --set profile=production -y

# Verify installation
kubectl get pods -n istio-system
kubectl get svc -n istio-system

# Expected output:
# istiod (control plane)
# istio-ingressgateway (ingress)
# istio-egressgateway (egress)
```

### Step 3: Enable Sidecar Injection

```bash
# Label namespace for automatic sidecar injection
kubectl label namespace production istio-injection=enabled

# Verify label
kubectl get namespace production --show-labels

# Expected: istio-injection=enabled
```

### Step 4: Deploy Test Application

```bash
# Create namespace
kubectl create namespace production

# Label for injection
kubectl label namespace production istio-injection=enabled

# Deploy httpbin test service (file lives in eks-kubernetes/samples/)
cd /home/frontier/devops-interview-prep/aws-devops
kubectl apply -f eks-kubernetes/samples/httpbin.yaml -n production

# Check pods have sidecar (2/2 containers = app + Envoy)
kubectl get pods -n production
kubectl describe pod -n production

# Expected: 2/2 containers per pod
```

### Step 5: Create Gateway and VirtualService

```bash
cat > gateway.yaml <<'EOF'
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: httpbin-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

---

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: httpbin
  namespace: production
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - route:
    - destination:
        host: httpbin
        port:
          number: 8000
EOF

kubectl apply -f gateway.yaml
```

### Step 6: Test Traffic Flow

```bash
# Port forward to ingress gateway
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80 &

# Test request
curl http://localhost:8080/status/200

# Expected: HTTP 200 response
```

### Verification Commands

```bash
# Check sidecar proxies
kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].name}'

# View Istio resources
kubectl get virtualservices -n production
kubectl get gateways -n production
kubectl get destinationrules -n production

# Check control plane
kubectl get pods -n istio-system
```

### Cleanup

```bash
kubectl delete ns production
istioctl uninstall --purge
```

**✓ Lab Complete:** Istio is installed and managing traffic!

---

## Lab 3: Implement Canary Deployment

**Duration:** 25-30 minutes
**Goal:** Deploy v2 as canary, gradually shift traffic

### Step 1: Deploy Stable Version (v1)

```bash
cat > v1-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-v1
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      version: v1
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
    spec:
      containers:
      - name: payment-service
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: SERVICE_VERSION
          value: "v1"
EOF

kubectl apply -f v1-deployment.yaml
kubectl get pods -n production
```

### Step 2: Deploy Canary Version (v2)

```bash
cat > v2-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-v2
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payment-service
      version: v2
  template:
    metadata:
      labels:
        app: payment-service
        version: v2
    spec:
      containers:
      - name: payment-service
        image: nginx:1.22
        ports:
        - containerPort: 80
        env:
        - name: SERVICE_VERSION
          value: "v2"
EOF

kubectl apply -f v2-deployment.yaml

# Verify both versions running
kubectl get pods -n production -L version
```

### Step 3: Create DestinationRule with Subsets

```bash
cat > destinationrule.yaml <<'EOF'
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: production
spec:
  host: payment-service.production.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF

kubectl apply -f destinationrule.yaml
```

### Step 4: Set Traffic Split 90/10

```bash
cat > virtualservice.yaml <<'EOF'
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
        subset: v2
      weight: 10
EOF

kubectl apply -f virtualservice.yaml
```

### Step 5: Monitor Canary Metrics

```bash
# Generate traffic
kubectl exec -it deployment/payment-service-v1 -n production -- \
  sh -c "while true; do curl http://localhost; sleep 1; done" &

# Check Envoy stats (request counts by subset)
kubectl logs -n production deploy/payment-service-v2 -c istio-proxy --tail=50 | grep route

# Simulate errors on v2 to test rollback
kubectl set env deployment/payment-service-v2 -n production ERROR_RATE=50
sleep 60
kubectl logs -n production deploy/payment-service-v2 --tail=20

# Check error rate
```

### Step 6: Progressive Traffic Shift

After 1 hour of monitoring:

```bash
# Shift to 25/75
kubectl patch vs payment-service -n production --type merge -p \
'{"spec":{"http":[{"route":[{"destination":{"host":"payment-service.production.svc.cluster.local","subset":"v1"},"weight":75},{"destination":{"host":"payment-service.production.svc.cluster.local","subset":"v2"},"weight":25}]}]}}'

# After another hour, shift to 50/50
# ... (same command, adjust weights)

# After final verification, shift to 0/100
kubectl patch vs payment-service -n production --type merge -p \
'{"spec":{"http":[{"route":[{"destination":{"host":"payment-service.production.svc.cluster.local","subset":"v1"},"weight":0},{"destination":{"host":"payment-service.production.svc.cluster.local","subset":"v2"},"weight":100}]}]}}'
```

### Step 7: Verify and Cleanup

```bash
kubectl delete deployment payment-service-v1 -n production
kubectl delete deployment payment-service-v2 -n production
```

**✓ Lab Complete:** You've successfully deployed a canary with gradual traffic shift!

---

## Lab 4: Troubleshoot Pod Networking Issues

**Duration:** 20-25 minutes
**Goal:** Debug common networking problems

### Create Test Setup

```bash
# Deploy two services
kubectl create namespace network-test
kubectl label namespace network-test istio-injection=enabled

cat > app1.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  namespace: network-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nicolaka/netshoot
        command: ["/bin/sleep", "3600"]

---

apiVersion: v1
kind: Service
metadata:
  name: app1-service
  namespace: network-test
spec:
  selector:
    app: app1
  ports:
  - port: 8080
    targetPort: 8080
EOF

kubectl apply -f app1.yaml
```

### Troubleshooting Scenarios

#### Scenario 1: DNS Resolution

```bash
# Enter pod
kubectl exec -it deployment/app1 -n network-test -- bash

# Test DNS resolution
nslookup app1-service
nslookup app1-service.network-test
nslookup app1-service.network-test.svc.cluster.local

# If fails, check CoreDNS:
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

#### Scenario 2: Network Connectivity

```bash
# Test connectivity
ping -c 2 app1-service.network-test.svc.cluster.local

# If fails, check iptables rules
kubectl debug node/$(kubectl get nodes -o name | head -1) -it --image=ubuntu
# Inside debug container:
iptables -t nat -L -n | grep app1
```

#### Scenario 3: Service Endpoints

```bash
# Check if service has endpoints
kubectl get endpoints -n network-test

# Expected output:
# NAME              ENDPOINTS
# app1-service      10.0.x.x:8080

# If empty, pods don't match selector:
kubectl get pods -n network-test --show-labels
kubectl describe service app1-service -n network-test
```

**✓ Lab Complete:** You can troubleshoot networking issues!

---

## Lab 5: Test Failure Scenarios and Recovery

**Duration:** 20-30 minutes
**Goal:** Simulate failures and observe automatic recovery

### Scenario 1: Pod Crash Recovery

```bash
# Deploy application
kubectl run nginx-app --image=nginx --replicas=3 -n default

# Simulate crash
POD=$(kubectl get pods -l run=nginx-app -o name | head -1)
kubectl delete $POD

# Observe automatic restart
kubectl get pods -l run=nginx-app --watch

# Expected: New pod created automatically
```

### Scenario 2: Node Drain (Planned Maintenance)

```bash
# Get a node
NODE=$(kubectl get nodes -o name | head -1)

# Drain the node (reschedules pods to other nodes)
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data

# Observe pods moving
kubectl get pods -n production --watch

# Once migration complete, uncordon node
kubectl uncordon $NODE
```

### Scenario 3: Pod Eviction (Resource Pressure)

```bash
# Create pods with resource requests/limits
cat > resource-test.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: 256Mi
      limits:
        memory: 256Mi

---

apiVersion: v1
kind: Pod
metadata:
  name: burstable
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: 256Mi
      limits:
        memory: 512Mi

---

apiVersion: v1
kind: Pod
metadata:
  name: besteffort
spec:
  containers:
  - name: app
    image: nginx
EOF

kubectl apply -f resource-test.yaml

# Simulate memory pressure by reducing available memory
# Observe which pods get evicted (besteffort first, then burstable)
```

**✓ Lab Complete:** You understand failure scenarios and recovery!

---

## Lab Verification Checklist

- [ ] Lab 1: Multi-environment Helm deployment working
- [ ] Lab 2: Istio installed with sidecars injected
- [ ] Lab 3: Canary deployment with traffic shifting successful
- [ ] Lab 4: Can troubleshoot DNS and networking issues
- [ ] Lab 5: Understand failure scenarios and recovery

