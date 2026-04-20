# 07 - Interview Q&A: EKS and Kubernetes Architecture

## Architecture and Design Questions

### Q1: "Describe your EKS architecture"

**2-3 minute answer:**

"I manage a multi-cluster EKS architecture across 4 environments: DEV, QA, Staging, and Production. Each cluster runs in its own VPC with a dedicated Kubernetes control plane managed by AWS.

**DEV Cluster:** Lightweight setup for rapid iteration. 1-2 t3.medium nodes, 3-4 namespaces, ~50-100 pods. Used for daily development and testing.

**QA Cluster:** Pre-production testing. 2-3 t3.large nodes, 5-8 namespaces, ~100-150 pods. Includes load testing and integration testing workloads.

**Staging Cluster:** Production mirror. 2-3 t3.large nodes with identical configuration to production. Used for final validation before production deployment.

**Production Cluster:** High-availability setup. 6-12 c5.xlarge nodes spread across 3 AZs for fault tolerance. Auto-scaling enabled to handle traffic spikes. 8-10 namespaces, 300-500+ pods. Critical services have 3+ replicas.

**Networking:** All clusters use VPC CNI, giving pods real VPC IPs. Each environment has its own VPC with custom subnets. Pods can directly communicate with AWS services (RDS, ElastiCache, S3) without NAT overhead.

**Traffic Management:** Istio service mesh deployed in all clusters for advanced traffic management, canary deployments, and mTLS between services. External traffic routed through AWS ALB via Kubernetes Ingress.

**Observability:** Centralized monitoring via Prometheus/Grafana for Kubernetes metrics, OpenTelemetry for distributed tracing, and CloudWatch for AWS-level metrics. All logs aggregated to CloudWatch Logs."

### Q2: "How do you implement canary deployments?"

**Answer:**

"I use Istio service mesh with weighted traffic routing. Here's my process:

1. **Deploy new version alongside stable:** I create a new Deployment with the new version (v2) while keeping the stable version (v1) running. Both are labeled to identify their versions.

2. **Create DestinationRule:** Define subsets based on version labels (v1 and v2).

3. **Create VirtualService with traffic split:** Initially, 90% traffic goes to v1, 10% to v2 (canary).

4. **Monitor canary metrics:** For 1 hour, I watch error rates, latency, and success rates. If canary error rate is < 0.1% and latency is similar to v1, proceed.

5. **Gradual traffic shift:** Hour 1: 10% → v2, Hour 2: 25% → v2, Hour 3: 50% → v2, Hour 4: 75% → v2, Hour 5: 100% → v2.

6. **Verify and finalize:** Once 100% traffic is on v2 and metrics are healthy for 1 hour, I delete the v1 deployment.

**Rollback:** If any issues are detected, I immediately set traffic back to 100% → v1. Istio handles the traffic shift at the proxy level, so it's instant with zero downtime."

### Q3: "What's the difference between Deployments and StatefulSets?"

**Answer:**

"Deployments are for stateless workloads where pods are interchangeable. StatefulSets are for stateful workloads where each pod has a stable identity and persistent storage.

**Deployments:**
- Pod names are random (payment-abc123)
- Pods are interchangeable (can be scheduled anywhere)
- Scaling up/down can happen in any order
- Good for microservices (API, payment processor, etc.)
- Use for: stateless web services, API servers

**StatefulSets:**
- Pod names are ordinal (postgres-0, postgres-1, postgres-2)
- Pods have stable identity and persistent storage
- Scaling maintains order (postgres-0 always has same storage)
- Good for databases, message queues, caching
- Use for: PostgreSQL, MongoDB, Redis, Kafka

Example: If I have a Deployment with 3 payment-service replicas, they're interchangeable. If I scale to 5, the 2 new pods are identical to existing ones. If I scale down to 1, any pod can be removed.

But if I have a StatefulSet with postgres-0, postgres-1, postgres-2, each has its own persistent volume. postgres-0 always uses volume-0. If I scale down to 1, postgres-2 and postgres-1 are removed in that order, preserving data."

### Q4: "How do resource requests and limits affect scheduling?"

**Answer:**

"Requests are used by the scheduler for placement decisions. Limits prevent pods from using more than that amount.

**Requests:** The scheduler looks at each node's available resources. If a node has 4 CPUs free and a pod requests 1 CPU, the scheduler can place the pod there. If a pod requests 4 CPUs but the node has only 2 free, the pod stays pending until a node with enough resources is available.

**Limits:** Once a pod is running, Kubernetes enforces limits. If a pod exceeds its memory limit, Kubernetes kills it and reschedules it. If CPU is exceeded, the container is throttled (limited to the CPU limit).

**QoS Classes:**
- **Guaranteed:** requests = limits. Pod won't be evicted.
- **Burstable:** requests < limits. Pod can be evicted if node has pressure.
- **BestEffort:** no requests/limits. Pod evicted first if node needs resources.

**My strategy:** For critical services, I set requests = expected average usage and limits = peak usage + 10% headroom. This ensures:
1. Scheduler makes good decisions (doesn't oversubscribe nodes)
2. Pods have room to burst during spikes
3. Pod won't be evicted due to another pod's spike
4. Node remains stable and responsive"

### Q5: "What happens when a node fails?"

**Answer:**

"When a node fails (hardware failure, power loss, network issue):

1. **Kubelet stops responding:** The node stops sending heartbeats to the control plane.

2. **After 5 minutes (default pod-eviction-timeout):** The control plane marks the node as NotReady and initiates pod eviction.

3. **Pods are rescheduled:** Kubernetes evicts all pods on the failed node and schedules them on healthy nodes. This typically takes 5-10 minutes.

4. **Data loss consideration:** Pods with emptyDir storage lose their data. Pods with PersistentVolumes can be recovered if the volume is still available (EBS/EFS).

5. **Replication saves you:** If a service has 3 replicas across 3 AZs and 1 AZ fails, the other 2 replicas continue serving traffic. Kubernetes reschedules the lost pod on a healthy node in another AZ.

**Best practices:**
- Always use multiple replicas for critical services
- Spread replicas across AZs (pod anti-affinity)
- Use Pod Disruption Budgets to maintain minimum availability during maintenance
- For stateful services (databases), use StatefulSets with persistent volumes

**In production:** If 1 node fails, users don't notice because other replicas handle the traffic. If multiple nodes fail simultaneously, we might lose some capacity, but data is preserved on PersistentVolumes."

### Q6: "How do you handle multi-tenant isolation in Kubernetes?"

**Answer:**

"I use namespaces, RBAC, network policies, and resource quotas:

1. **Namespaces:** Each team gets its own namespace (team-payments, team-users, team-orders). Namespaces provide logical isolation and are a natural boundary for RBAC and resource quotas.

2. **RBAC:** Each team's service account can only access resources in their namespace. The CI/CD system authenticates as a specific service account with minimal permissions. This prevents one team from accidentally modifying another team's deployments.

3. **Network Policies:** I create policies that deny all traffic by default (deny-all policy), then allow only necessary communication. For example, team-payments services can only communicate with each other and the database, not with team-users services.

4. **Resource Quotas:** Each namespace has a resource quota limiting total CPU, memory, and pod count. This prevents one team from consuming all cluster resources, impacting other teams.

5. **Pod Security Policies:** All pods must run as non-root, drop all capabilities, and cannot mount host paths. This prevents privilege escalation attacks.

**Example setup:**
```
Namespace: team-payments
├── ResourceQuota: 20 CPUs, 40Gi memory, 100 pods
├── NetworkPolicy: Only allow ingress from ingress-gateway
└── RBAC: team-payments-sa can only deploy/update in this namespace

Namespace: team-users
├── ResourceQuota: 15 CPUs, 30Gi memory, 80 pods
├── NetworkPolicy: Only allow ingress from ingress-gateway
└── RBAC: team-users-sa can only deploy/update in this namespace
```

This isolation ensures teams don't interfere with each other while sharing the same cluster efficiently."

---

## Istio and Traffic Management Questions

### Q7: "Explain Istio and why you use it"

**Answer:**

"Istio is a service mesh that provides sophisticated traffic management, security, and observability without requiring application code changes. It works by injecting Envoy proxy sidecars into every pod.

**Why I use it:**

1. **Canary deployments:** Traffic shifting lets me gradually move traffic from old to new versions with automatic rollback if issues occur.

2. **Traffic management:** Retry policies, timeouts, circuit breakers, and rate limiting at the infrastructure level, not in app code.

3. **mTLS:** All pod-to-pod traffic is automatically encrypted with mutual TLS. Certificates are automatically rotated. Zero trust security model.

4. **Observability:** Distributed tracing (who calls whom), golden metrics (latency, error rates), and automatic service dependency graphs.

5. **Resilience:** Outlier detection automatically removes unhealthy pods from load balancing. Circuit breakers prevent cascading failures.

**Cost:** Istio adds ~10-15% overhead per pod (Envoy sidecar resources), but it prevents bugs, makes deployments safer, and improves reliability."

### Q8: "How do VirtualServices and DestinationRules work?"

**Answer:**

"VirtualService defines routing rules for traffic. DestinationRule defines how to handle traffic once routed.

**VirtualService (how to route):**
```
- Match: request to /payments/transfer
- Route to: payment-service with 100% traffic
- Retry: 3 attempts
- Timeout: 30 seconds
```

**DestinationRule (how to handle):**
```
- Host: payment-service
- Load balancer: ROUND_ROBIN
- Connection pool: Max 100 connections
- Outlier detection: Eject pods with 5 consecutive errors
- mTLS: STRICT (require encryption)
- Subsets: v1, v2 based on labels
```

**Together:** VirtualService says 'send 90% to v1, 10% to v2'. DestinationRule defines that v1 pods are those labeled version=v1. It also defines that load balancing should use ROUND_ROBIN and pods with too many errors should be ejected."

---

## Failure and Troubleshooting Questions

### Q9: "A pod is crashing in production. How do you troubleshoot?"

**Answer:**

"Step 1: Check pod status
```
kubectl describe pod <name> -n production
```
Look for: container state, exit code, events

Step 2: View logs
```
kubectl logs <pod> -n production --previous  # If crashed
kubectl logs <pod> -n production  # Current logs
```

Step 3: If still unclear, exec into pod
```
kubectl exec -it <pod> -n production -- bash
```

Step 4: Check dependencies
- Database connectivity: Can pod reach database?
- Secrets: Are database credentials correct?
- DNS: Can pod resolve service names?

Step 5: Check resource constraints
```
kubectl top pod -n production
```
Are we hitting memory/CPU limits?

Step 6: Check readiness probe
```
kubectl describe deployment <name> -n production
```
Is the health check endpoint working?

Step 7: Check node health
```
kubectl describe node <node-name>
```
Is the node running out of resources?"

### Q10: "Service is returning 503 errors. How do you debug?"

**Answer:**

"503 means the service is unavailable. Possible causes:

1. **All pods are failing:** Check pod logs for app errors.
2. **No pods are ready:** Check readiness probe - is the health check endpoint working?
3. **Network connectivity:** Can the ingress reach the pods? Check security groups and network policies.
4. **Resource exhaustion:** Are pods hitting CPU/memory limits and being throttled?

Debugging steps:
```
# Check endpoints
kubectl get endpoints <service> -n production

# If empty, no pods match selector
kubectl get pods -n production --show-labels

# Check events
kubectl describe service <name> -n production

# Check pod logs
kubectl logs <pod> -n production

# Test connectivity
kubectl exec -it <pod> -n production -- curl localhost:8080/health

# Check CPU/memory usage
kubectl top pod -n production
```



## Monitoring and Observability Questions

### Q11: "How do you monitor EKS clusters?"

**Answer:**

"I use a multi-layer monitoring strategy:

1. **Kubernetes Metrics (Prometheus/Grafana):**
   - Pod CPU/memory usage
   - Node capacity and utilization
   - Deployment replica counts
   - Service request rates

2. **AWS CloudWatch:**
   - EKS control plane metrics (API latency, request counts)
   - EKS node group status
   - Worker node EC2 metrics (disk, network)
   - CloudWatch alarms for critical metrics

3. **Application Metrics (OpenTelemetry):**
   - Request latency, error rates, success rates
   - Business metrics (payments processed, orders created)
   - Custom application metrics

4. **Distributed Tracing:**
   - Trace requests across microservices
   - Identify bottlenecks
   - Debug performance issues

5. **Logging:**
   - Application logs to CloudWatch Logs
   - Kubernetes audit logs
   - Pod logs aggregated for searching

**SLOs/SLIs:**
- SLI: 99.95% of requests complete within 500ms
- SLI: 99.9% availability (max 43 seconds downtime/month)
- Error budget: If we exceed error budget, no new deployments until recovery"

```
```
## Best Practices Questions

### Q12: "What are your Kubernetes best practices?"

**Answer:**

"**Resource Management:**
- Always set requests and limits
- Use requests = expected usage, limits = peak usage
- Monitor actual usage and adjust

**High Availability:**
- 3+ replicas for critical services
- Spread replicas across AZs (pod anti-affinity)
- Pod Disruption Budgets for graceful disruptions
- Multiple node groups across AZs

**Security:**
- No privileged containers
- Run as non-root
- Enable RBAC, network policies
- Scan images for vulnerabilities
- Use Secrets Manager for sensitive data
- Enable audit logging

**Observability:**
- Health checks (liveness + readiness)
- Structured logging (JSON format)
- Distributed tracing
- Prometheus metrics
- Gradual rollouts (canary deployments)

**Operations:**
- Infrastructure as Code (Terraform, Helm)
- Automated testing in CI/CD
- Automated deployments
- Regular backups (etcd, persistent data)
- Disaster recovery drills

**Cost Optimization:**
- Right-size instances (don't over-provision)
- Use spot instances for non-critical workloads
- Implement resource quotas per namespace
- Monitor and alert on cost anomalies"

```

