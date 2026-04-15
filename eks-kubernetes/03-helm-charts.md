# 03 - Helm Charts: Package Management for Kubernetes

## What is Helm?

Helm is the package manager for Kubernetes, like npm for Node.js or pip for Python. It simplifies deploying complex applications with multiple Kubernetes resources (Deployments, Services, ConfigMaps, Secrets, etc.).

### Problem Helm Solves

```
Without Helm:
- Create 10+ YAML files manually
- Different values for dev/staging/prod
- Duplicate similar YAML files
- Hard to manage versions
- Manual install/upgrade process

With Helm:
- Single chart package
- Templated values for all environments
- Version control built-in
- One-command install/upgrade
- Dependency management
```

---

## Helm Chart Structure

```
my-payment-service-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── values-dev.yaml         # Dev overrides
├── values-prod.yaml        # Prod overrides
├── templates/
│   ├── deployment.yaml     # Templated Deployment
│   ├── service.yaml        # Templated Service
│   ├── ingress.yaml        # Templated Ingress
│   ├── configmap.yaml      # Templated ConfigMap
│   ├── secret.yaml         # Templated Secret
│   ├── hpa.yaml            # Horizontal Pod Autoscaler
│   ├── pdb.yaml            # Pod Disruption Budget
│   ├── _helpers.tpl        # Template helpers
│   └── NOTES.txt           # Post-install instructions
├── Chart.lock              # Locked dependencies
└── charts/                 # Dependent charts
    └── postgres/           # PostgreSQL subchart
```

---

## Chart.yaml: Chart Metadata

```yaml
apiVersion: v2                    # Helm 3
name: payment-service
description: Payment processing microservice
type: application                 # vs 'library'
version: 1.5.0                    # Chart version (not app version!)
appVersion: "2.3.1"               # Application version
keywords:
  - payment
  - microservice
  - go
home: https://github.com/company/payment-service
sources:
  - https://github.com/company/payment-service
maintainers:
  - name: Platform Team
    email: platform@company.com
dependencies:                     # Subchart dependencies
  - name: postgresql
    version: "12.1.2"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
    alias: postgres
```

---

## values.yaml: Configuration Template

```yaml
# Default values for payment-service chart

# Global settings
global:
  environment: production
  domain: example.com
  imageRegistry: myregistry.azurecr.io

# Image configuration
image:
  repository: payment-service
  tag: "2.3.1"
  pullPolicy: IfNotPresent

# Deployment configuration
replicaCount: 3

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# Pod settings
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"
  prometheus.io/path: "/metrics"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL

# Resource requests and limits
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi

# Auto-scaling
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Service configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  annotations: {}

# Ingress configuration
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  hosts:
    - host: api.example.com
      paths:
        - path: /v1/payments
          pathType: Prefix
  tls:
    - secretName: payment-tls
      hosts:
        - api.example.com

# Environment variables
env:
  LOG_LEVEL: "INFO"
  FEATURES_ENABLED: "payment-v2"
  DATABASE_POOL_SIZE: "20"

# Secrets (injected from AWS Secrets Manager)
secrets:
  DATABASE_URL: "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-url"
  API_KEY: "arn:aws:secretsmanager:us-east-1:123456789012:secret:api-key"

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 2

# Database configuration (for postgres subchart)
postgresql:
  enabled: true
  auth:
    username: appuser
    password: changeme
    database: payment
  primary:
    persistence:
      size: 100Gi
      storageClassName: ebs-gp3

# Monitoring
monitoring:
  prometheus:
    enabled: true
    interval: 30s
  alerts:
    enabled: true
```

---

## Templated Deployment (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "payment-service.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "payment-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "payment-service.selectorLabels" . | nindent 6 }}
  strategy:
    {{- toYaml .Values.strategy | nindent 4 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "payment-service.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    spec:
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "payment-service.serviceAccountName" . }}
      containers:
      - name: {{ .Chart.Name }}
        {{- with .Values.securityContext }}
        securityContext:
          {{- toYaml . | nindent 12 }}
        {{- end }}
        image: "{{ .Values.global.imageRegistry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        envFrom:
        - configMapRef:
            name: {{ include "payment-service.fullname" . }}
        - secretRef:
            name: {{ include "payment-service.fullname" . }}
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        {{- with .Values.resources }}
        resources:
          {{- toYaml . | nindent 12 }}
        {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

**Template syntax:**
```
{{ .Values.key }}               - Reference values
{{ .Chart.AppVersion }}         - Chart metadata
{{ include "helper" . }}        - Call template helper
{{ range .Values.items }}       - Loop over items
{{- if condition }}             - Conditional (- removes whitespace)
{{ toYaml .Values.obj }}        - Convert to YAML
```

---

## Helm Commands

### Install

```bash
# Install from local chart
helm install payment-service ./payment-service-chart \
  -n production \
  --create-namespace \
  -f values-prod.yaml

# Install from repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-postgres bitnami/postgresql \
  --version 12.1.2 \
  -n databases

# Install with inline values
helm install payment-service ./payment-service-chart \
  --set replicaCount=5 \
  --set image.tag=v2.4.0 \
  --set autoscaling.maxReplicas=20
```

### Upgrade

```bash
# Upgrade to new version
helm upgrade payment-service ./payment-service-chart \
  -n production \
  -f values-prod.yaml

# Upgrade and reuse values from previous release
helm upgrade payment-service ./payment-service-chart \
  -n production \
  --reuse-values

# Upgrade with new values
helm upgrade payment-service ./payment-service-chart \
  -n production \
  --values values-prod.yaml \
  --set image.tag=v2.5.0
```

### Rollback

```bash
# Rollback to previous release
helm rollback payment-service \
  -n production

# Rollback to specific revision
helm rollback payment-service 3 \
  -n production

# View release history
helm history payment-service -n production
```

### Inspect

```bash
# List installed releases
helm list -n production

# Get release values
helm get values payment-service -n production

# View generated manifests
helm template payment-service ./payment-service-chart \
  -f values-prod.yaml

# Validate chart
helm lint ./payment-service-chart

# Check for issues
helm get values payment-service -n production
helm get manifest payment-service -n production
```

---

## Release Strategies with Helm

### Canary Deployment using Helm + Istio

```yaml
# values-canary.yaml
image:
  tag: v2.5.0-canary
replicaCount: 1  # 1 canary pod

# In Istio VirtualService (separate manifest)
# - 10% traffic to canary (v2.5.0)
# - 90% traffic to stable (v2.4.0)
```

**Deploy:**
```bash
# Deploy canary version
helm upgrade payment-service ./chart \
  -n production \
  -f values-prod.yaml \
  -f values-canary.yaml

# Check canary metrics (in parallel with stable)
# Monitor: error rate, latency, success rate

# If good: promote canary to stable
helm upgrade payment-service ./chart \
  -n production \
  -f values-prod.yaml \
  --set image.tag=v2.5.0 \
  --set replicaCount=3

# If bad: rollback
helm rollback payment-service -n production
```

---

## Common Interview Questions

### Q: "What problem does Helm solve?"

**Answer:**
"Helm solves the problem of managing complex Kubernetes deployments across multiple environments. Without Helm, I'd have 15-20 YAML files and manually change values for dev/staging/prod. With Helm, I have one templated chart and just override values. Helm also handles version management, dependencies, and one-command installs/upgrades/rollbacks. It's especially powerful for package management - I can reuse charts across projects and teams."

### Q: "How do you structure values for multiple environments?"

**Answer:**
"I use a base values.yaml with sensible defaults, then create environment-specific overrides: values-dev.yaml, values-staging.yaml, values-prod.yaml. Each contains only the differences from base. When installing, I stack them: `helm install -f values.yaml -f values-prod.yaml`. This layers the values: base + prod overrides. For example, prod has 3 replicas and larger resources, dev has 1 replica and smaller resources."

### Q: "Explain templating in Helm charts."

**Answer:**
"Helm uses Go templating with Sprig functions. I can reference values with `{{ .Values.key }}`, loop with `{{ range }}`, use conditionals with `{{ if }}`, and call helpers with `{{ include }}`. For example, in my Deployment template, I reference `{{ .Values.replicaCount }}` and `{{ .Values.image.tag }}`. Helm processes these before applying to Kubernetes. This lets me use one template for multiple scenarios."

