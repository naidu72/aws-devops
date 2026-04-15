# 05 - Ingress and Load Balancing: External Traffic Management

## AWS Load Balancer Controller

AWS Load Balancer Controller for Kubernetes creates AWS ALB/NLB resources from Kubernetes Ingress manifests.

### Install AWS Load Balancer Controller

```bash
# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=true \
  --set enableShield=false \
  --set enableWaf=false \
  --set enableWafv2=false
```

---

## Ingress: Application Load Balancer (ALB)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip  # Use pod IPs directly
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc123
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/actions.ssl-redirect: |
      {
        "Type": "redirect",
        "RedirectConfig": {
          "Protocol": "HTTPS",
          "Port": "443",
          "StatusCode": "HTTP_301"
        }
      }
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
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
      - path: /v1/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
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
  tls:
  - hosts:
    - api.example.com
    - admin.example.com
    secretName: api-tls-cert
```

**What this creates:**
```
1. ALB in AWS (internet-facing)
2. Rules:
   - api.example.com/v1/users → user-service (port 80)
   - api.example.com/v1/payments → payment-service (port 80)
   - admin.example.com/* → admin-service (port 80)
3. HTTPS listener (443) with certificate
4. HTTP listener (80) with redirect to HTTPS
5. Health checks pointing to /health
6. Target groups mapping to Kubernetes pods
```

---

## Network Load Balancer (NLB) vs ALB

### ALB (Application Load Balancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    # Actually uses ALB for Ingress (not NLB service)
```

**Use ALB for:**
- HTTP/HTTPS traffic (Layer 7)
- Path-based routing (/api, /admin)
- Host-based routing (api.example.com)
- Multiple services through single LB
- Cost: ~$18-25/month per ALB

---

### NLB (Network Load Balancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service-nlb
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: payment-service
  ports:
  - protocol: TCP
    port: 443
    targetPort: 8080
```

**Use NLB for:**
- TCP/UDP traffic (Layer 4)
- Ultra-high throughput (millions of requests/sec)
- Low latency requirements
- Non-HTTP protocols (gRPC, custom binary)
- Cost: ~$36-50/month per NLB (2x more expensive than ALB)

---

## Service Discovery: DNS

### In-Cluster DNS (CoreDNS)

```bash
# Pod can resolve services via DNS
kubectl run -it debug --image=busybox -- sh

# Within the pod:
nslookup payment-service              # finds payment-service.default.svc.cluster.local
nslookup payment-service.production   # finds payment-service.production.svc.cluster.local
nslookup payment-service.production.svc.cluster.local
```

### DNS Query Flow

```
Pod A container resolves: payment-service
           ↓
Pod's /etc/resolv.conf:
  nameserver 10.100.0.10  (CoreDNS)
           ↓
Query: payment-service.production.svc.cluster.local
           ↓
CoreDNS lookup (in kube-system namespace)
           ↓
Returns: Service ClusterIP 10.100.0.42
           ↓
Pod connects to 10.100.0.42:80
           ↓
kube-proxy iptables rule routes to pod IPs
           ↓
Connection to actual pod
```

### External DNS

For external DNS (route53), use ExternalDNS:

```yaml
apiVersion: external-dns.alpha.k8s.io/v1alpha1
kind: DNSEndpoint
metadata:
  name: payment-service
  namespace: production
spec:
  endpoints:
  - dnsName: api.example.com
    recordTTL: 300
    recordType: A
    targets:
    - alb-12345.us-east-1.elb.amazonaws.com
```

---

## Ingress vs Service

### When to Use What

| Type | Use Case | Cost |
|------|----------|------|
| Service ClusterIP | Pod-to-pod communication | Free |
| Service NodePort | Development/testing | Free |
| Service LoadBalancer | External access to single service | $18-50/month |
| Ingress ALB | Multiple services, HTTP/HTTPS | $18-25/month (shared) |
| Ingress NLB | High throughput, low latency | $36-50/month |

### Rule of Thumb

```
1-2 services needing external access → Use Service LoadBalancer

3+ services needing external access → Use Ingress ALB
  (1 ALB can handle all with path/host routing)

High-frequency, low-latency traffic → Use Ingress NLB or Service NLB

Internal pod communication → Use Service ClusterIP
```

---

## Health Checks and Routing

### Kubernetes Health Probes

```yaml
containers:
- name: app
  ports:
  - name: http
    containerPort: 8080
  livenessProbe:
    httpGet:
      path: /health/live
      port: http
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health/ready
      port: http
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 2
```

**Liveness:** Is the pod still alive? If fails 3 times, restart pod.
**Readiness:** Is the pod ready to serve traffic? If fails, remove from service endpoints.

### ALB Health Checks

```yaml
annotations:
  alb.ingress.kubernetes.io/healthcheck-path: /health/ready
  alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
  alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
  alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
  alb.ingress.kubernetes.io/healthy-threshold-count: '2'
  alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
```

**ALB flow:**
```
Every 15 seconds:
- ALB sends GET /health/ready to pod
- Expects HTTP 200 response within 5 seconds
- If 2 consecutive successes: mark as healthy
- If 2 consecutive failures: mark as unhealthy (remove from rotation)
```

---

## Advanced Routing

### Path-Based Routing

```yaml
rules:
- host: api.example.com
  http:
    paths:
    - path: /v1/users(/|$)(.*)
      pathType: Prefix
      backend:
        service:
          name: user-service
          port:
            number: 80
    - path: /v1/payments(/|$)(.*)
      pathType: Prefix
      backend:
        service:
          name: payment-service
          port:
            number: 80
```

### Host-Based Routing

```yaml
rules:
- host: api.example.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: api-service
- host: admin.example.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: admin-service
- host: webhook.example.com
  http:
    paths:
    - path: /
      backend:
        service:
          name: webhook-service
```

---

## Common Interview Questions

### Q: "Why use Ingress instead of LoadBalancer Service?"

**Answer:**
"LoadBalancer Service creates a dedicated AWS load balancer (~$18-50/month) for each service. If I have 5 services needing external access, that's 5 load balancers, costing hundreds per month. Ingress lets me use a single ALB to route to multiple services via path or host-based rules. One ALB costs $18-25/month and handles unlimited services. So with 5 services: LoadBalancer costs ~$100/month, Ingress costs ~$20/month. Ingress also provides Layer 7 routing, SSL termination, and more sophisticated rules."

### Q: "How do health checks work between Kubernetes and ALB?"

**Answer:**
"Kubernetes has readiness probes - if a pod fails readiness, Kubernetes removes it from the Service's endpoint list. ALB also does its own health checks: every 15 seconds, it sends requests to pods. If a pod fails 2 consecutive checks, ALB removes it from the target group. Both work together: Kubernetes readiness prevents traffic from routing to failing pods, and ALB health checks catch issues that Kubernetes might miss. They're redundant but that's good - defense in depth."

### Q: "What's the difference between ALB and NLB?"

**Answer:**
"ALB is Layer 7 (Application): it understands HTTP/HTTPS and can route based on URLs, paths, hostnames. NLB is Layer 4 (Transport): it's just TCP/UDP and super fast but doesn't understand HTTP. ALB costs less and is fine for web services. I'd use NLB for APIs needing ultra-low latency or non-HTTP protocols like gRPC. Most of my services use ALB with path-based routing."

