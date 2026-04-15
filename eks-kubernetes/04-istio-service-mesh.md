# 04 - Istio Service Mesh: Advanced Traffic Management

## What is Istio?

Istio is a service mesh that provides sophisticated traffic management, security, and observability for microservices without requiring changes to application code. It uses sidecar proxies (Envoy) injected into every pod.

### Architecture

```
┌──────────────────────────────────────────────────────┐
│              Kubernetes Cluster                      │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Payment Service Pod                          │ │
│  │  ┌──────────────────────┐  ┌──────────────┐  │ │
│  │  │  app container       │  │ Envoy proxy  │  │ │
│  │  │  (payment-service)   │  │ (sidecar)    │  │ │
│  │  │  :8080               │  │ :15000       │  │ │
│  │  └──────────────────────┘  └──────────────┘  │ │
│  │  All traffic flows through Envoy proxy!      │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  User Service Pod                             │ │
│  │  ┌──────────────────────┐  ┌──────────────┐  │ │
│  │  │  app container       │  │ Envoy proxy  │  │ │
│  │  │  (user-service)      │  │ (sidecar)    │  │ │
│  │  │  :8080               │  │ :15000       │  │ │
│  │  └──────────────────────┘  └──────────────┘  │ │
│  └────────────────────────────────────────────────┘ │
│         ↓ (Envoy-to-Envoy mTLS)                    │
│  ┌────────────────────────────────────────────────┐ │
│  │  Order Service Pod                            │ │
│  │  ┌──────────────────────┐  ┌──────────────┐  │ │
│  │  │  app container       │  │ Envoy proxy  │  │ │
│  │  │  (order-service)     │  │ (sidecar)    │  │ │
│  │  │  :8080               │  │ :15000       │  │ │
│  │  └──────────────────────┘  └──────────────┘  │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌──────────────────────────────────────────────┐   │
│  │  Istiod Control Plane (kube-system)         │   │
│  │  - Manages all Envoy proxies                │   │
│  │  - Updates routing rules                    │   │
│  │  - Handles certificates (mTLS)             │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

---

## VirtualService: Routing Rules

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
  namespace: production
spec:
  hosts:
  - payment-service.production.svc.cluster.local  # Kubernetes DNS
  http:
  # Canary: 10% traffic to v2, 90% to v1
  - match:
    - uri:
        prefix: /v2/payments
    route:
    - destination:
        host: payment-service.production.svc.cluster.local
        subset: v2
        port:
          number: 8080
  - route:
    - destination:
        host: payment-service.production.svc.cluster.local
        subset: v1
        port:
          number: 8080
        weight: 90
    - destination:
        host: payment-service.production.svc.cluster.local
        subset: v2
        port:
          number: 8080
        weight: 10
    timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
```

---

## DestinationRule: Subsets and Load Balancing

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
        h2UpgradePolicy: UPGRADE
    loadBalancer:
      simple: ROUND_ROBIN  # or LEAST_REQUEST, RANDOM, PASSTHROUGH
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minRequestVolume: 5
    tls:
      mode: ISTIO_MUTUAL  # mTLS encryption
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: canary
    labels:
      version: v2-canary
    trafficPolicy:
      connectionPool:
        http:
          maxRequestsPerConnection: 5
```

**What this does:**
- `subsets`: Divide traffic by pod labels (v1, v2, canary)
- `loadBalancer`: ROUND_ROBIN distributes evenly
- `outlierDetection`: Remove pods that fail 5 consecutive requests
- `connectionPool`: Limit connections to prevent overload
- `tls`: Enforce mTLS (encrypted pod-to-pod traffic)

---

## Gateway: External Traffic Entry Point

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: payment-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: payment-tls-cert  # Secret with cert
    hosts:
    - "api.example.com"
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "api.example.com"
    # HTTP redirect to HTTPS
    httpsRedirect: true

---

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-api
  namespace: production
spec:
  hosts:
  - "api.example.com"  # External hostname
  gateways:
  - payment-gateway
  http:
  - route:
    - destination:
        host: payment-service.production.svc.cluster.local
        port:
          number: 8080
```

**Traffic flow:**
```
User: https://api.example.com/v1/payments
    ↓
Ingress Gateway (Envoy at edge)
    ↓
Matches Gateway rules (port 443, host: api.example.com)
    ↓
Routes via VirtualService
    ↓
payment-service pods (via DestinationRule)
```

---

## Canary Deployment with Istio

### Step 1: Deploy stable version

```bash
kubectl apply -f deployment-v1.yaml  # v1, label: version=v1
```

### Step 2: Deploy canary version alongside

```bash
kubectl apply -f deployment-v2-canary.yaml  # v2, label: version=v2-canary
```

### Step 3: Split traffic 90/10

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
        subset: stable
      weight: 90
    - destination:
        host: payment-service
        subset: canary
      weight: 10

---

apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  subsets:
  - name: stable
    labels:
      version: v1
  - name: canary
    labels:
      version: v2-canary
```

### Step 4: Monitor canary metrics

```
Monitor for 1 hour:
- Error rate (should be < 0.1%)
- Latency (should be similar to v1)
- Success rate (should be 99.9%+)

If all good: Increase to 50/50
```

### Step 5: Gradual rollout

```
Hour 1: 10% → v2
Hour 2: 25% → v2
Hour 3: 50% → v2
Hour 4: 75% → v2
Hour 5: 100% → v2
```

### Step 6: Delete v1, fully promote v2

```bash
kubectl delete deployment payment-service-v1
helm upgrade payment-service ./chart --set image.tag=v2
```

---

## Traffic Policies: Advanced Features

### Rate Limiting

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
    rateLimit:
      actions:
      - quota:
          - quota: payment-api-quota
            unit: minute
            value: 1000
```

### Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5        # Eject after 5 errors
      interval: 30s                  # Check every 30s
      baseEjectionTime: 30s          # Min ejection duration
      maxEjectionPercent: 50         # Max 50% of pods ejected
      minRequestVolume: 5            # Need 5 requests to trigger
```

### Retry Policy

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
    retries:
      attempts: 3                # Retry 3 times
      perTryTimeout: 10s        # 10s per retry
    timeout: 30s                # Total timeout 30s
```

---

## mTLS: Mutual TLS Encryption

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS for all pods
```

**What happens:**
```
Pod A → Pod B

Before mTLS:
Pod A sends plain HTTP:
  User-Agent: curl
  GET /api/users

After mTLS:
Pod A (Envoy) → Pod B (Envoy):
  TLS encrypted
  Certificates exchanged
  Automatic renewal by Istiod

Result:
- All pod-to-pod traffic encrypted
- No app code changes needed
- Automatic certificate rotation
- Zero-trust security model
```

---

## Common Interview Questions

### Q: "Why use Istio instead of Kubernetes Ingress?"

**Answer:**
"Kubernetes Ingress handles Layer 7 external traffic, but only between external users and your cluster. Istio handles all traffic: external and internal. Ingress can't do canary deployments, circuit breaking, or mTLS. With Istio, I get sophisticated traffic management (weighted routing, retries, timeouts), canary deployments with automatic traffic shifting, automatic mTLS between all pods, and advanced observability (distributed tracing). Istio is more complex, but for a production microservices platform, the features are worth it."

### Q: "How do you implement canary deployments with Istio?"

**Answer:**
"I deploy both versions (v1 and v2) simultaneously, each with different pod labels. I create a DestinationRule that defines subsets based on these labels. Then I create a VirtualService that splits traffic: 90% to v1 subset, 10% to v2 subset. I monitor the canary version's metrics (error rate, latency). If healthy after 1 hour, I increase to 25%. If any issues, I immediately roll back by setting weight to 0% for v2. Once fully verified (usually 4-5 hours), I scale down v1 and complete the rollout."

### Q: "Explain mTLS in Istio."

**Answer:**
"mTLS (Mutual TLS) means every pod verifies the identity of every other pod it communicates with using certificates. In Istio, this happens at the Envoy sidecar layer, so applications don't need code changes. Istiod manages certificate lifecycle: issuance, rotation, distribution. Every pod gets a unique certificate with identity information. When Pod A communicates with Pod B, both Envoy proxies verify each other's certificates before establishing the connection. All traffic is encrypted. I enable strict mTLS mode to ensure all traffic must be encrypted."

### Q: "How do outlier detection and circuit breakers work?"

**Answer:**
"Outlier detection removes unhealthy pods from the load balancing pool automatically. I configure it to eject a pod after 5 consecutive 5xx errors, remove it for 30 seconds, then try again. maxEjectionPercent prevents ejecting all pods at once. This acts like a circuit breaker - if a pod is misbehaving, stop sending traffic to it, giving it time to recover. Combined with retries, this improves resilience: if a pod is ejected, retries go to healthy pods."

