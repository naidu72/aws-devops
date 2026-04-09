# Kubernetes Interview Preparation - Execution Guide

## Part 1: Core K8s Resources (Week 1-2)

### Study Resources:
1. **Pod Fundamentals**
   - Review: Ephemeral nature, QoS classes, init containers, sidecar patterns
   - Practice: Create pods with resource requests/limits
   - Interview Q: "What are init containers used for?"

2. **Deployment Deep Dive**
   - Review: Rolling updates, MaxSurge/MaxUnavailable, revision history, rollback
   - Workspace Example: hansen-cloud-native-apps/apps/cpq/helm/cpq/templates/deployment.yaml
   - Practice: Design deployment strategy for zero-downtime updates

3. **StatefulSet vs Deployment**
   - Review: Ordinal indices, stable network IDs, PersistentVolumeClaim binding
   - Workspace Example: hansen-cloud-native-suite/eksConfig/modules/k8s/storage/
   - Practice: Deploy stateful database with persistent storage

4. **DaemonSet, Job, CronJob**
   - Review: Node affinity, taints/tolerations, batch task patterns
   - Practice: Create monitoring agent as DaemonSet

5. **Service & Ingress**
   - Review: ClusterIP, NodePort, LoadBalancer, service discovery DNS
   - Workspace Example: hansen-cloud-native-apps/apps/cs/helm/cs-core/templates/service.yaml
   - Practice: Design multi-tier networking

### Assessment:
- [ ] Can explain when to use each resource type
- [ ] Can design deployment strategy for different scenarios
- [ ] Can troubleshoot pod scheduling issues

---

## Part 2: Storage & Volume Management (Week 2)

### Key Topics:
1. PersistentVolume (PV) & PersistentVolumeClaim (PVC)
2. Volume Types (EBS, EFS, emptyDir, hostPath)
3. StorageClass for dynamic provisioning
4. StatefulSet + PVC integration

### Workspace Deep Dive:
- Study: hansen-cloud-native-suite/eksConfig/modules/k8s/storage/

### Practice Scenario:
- Deploy PostgreSQL with persistent storage
- Test: Verify data persists after pod deletion

---

## Part 3: Networking Deep Dive (Week 2-3)

### CNI Plugins:
1. **Amazon VPC CNI** (workspace focus)
   - WARM_IP_TARGET tuning
   - Security group integration
   - Study: hansen-cloud-native-suite/eksv2/modules/eksCluster/main.tf

2. **Calico vs Cilium**
   - eBPF vs iptables
   - Network policy enforcement

### Service Discovery:
- CoreDNS internals
- kube-proxy modes
- Endpoint slices

### Practice:
- Write NetworkPolicies for multi-tier apps
- Troubleshoot connectivity issues

---

## Part 4: Platform Architecture (Week 3-4)

### EKS Deep Dive:
- Control plane managed service
- Node groups & Launch templates
- IRSA (IAM Roles for Service Accounts)
- Study: hansen-cloud-native-suite/eksv2/modules/eksCluster/main.tf

### AKS Architecture:
- Managed identity integration
- Azure CNI setup

### Bare Metal:
- kubeadm bootstrap
- etcd HA setup
- Storage options

---

## Part 5: Monitoring & Observability (Week 4-5)

### Prometheus & Metrics:
- PromQL queries
- ServiceMonitor CRDs
- Workspace: Monitoring stack examples

### Logging Stack:
- Fluent-bit collection
- Elasticsearch/Loki storage
- Kibana/Grafana visualization

### Distributed Tracing:
- Jaeger deployment
- OpenTelemetry instrumentation

---

## Part 6: Cloud-Native Tools (Week 6-7)

### AWS:
- Secrets Manager & Parameter Store
- KMS encryption
- CloudWatch monitoring
- X-Ray tracing
- GuardDuty security

### Azure:
- Key Vault
- Application Insights
- Log Analytics
- Defender for Cloud

### Multi-Cloud:
- HashiCorp Vault setup
- External Secrets Operator
- Secret rotation strategies

---

## Part 7: Hands-On Practicals (Week 7-9)

### Complete These Practicals:
1. ✓ StatefulSet with persistence
2. ✓ NetworkPolicy zero-trust
3. ✓ Helm chart troubleshooting
4. ✓ HPA scaling design
5. ✓ Multi-arch image builds
6. ✓ Cluster backup/restore
7. ✓ Complete observability stack
8. ✓ Security scanning pipeline
9. ✓ Multi-cluster monitoring

---

## Part 8: Mock Interviews (Week 10)

### Interview Scenarios:
1. Design Kubernetes platform (startup context)
2. Debug production issue (latency)
3. Bare metal cluster failure response
4. Multi-cloud deployment strategy
5. Secret rotation across environments

---

## Part 9: Interview Questions Bank

### Categories:
- Core Resources (Q1-Q3)
- Networking (Q4-Q5)
- Advanced Topics (Q6-Q7)
- Prometheus (Q8-Q10)
- Grafana (Q11-Q12)
- Logging (Q13-Q14)
- Tracing (Q15-Q16)
- Service Mesh (Q17-Q18)
- Security (Q19-Q22)
- Cloud Native (Q23-Q26)

---

## Study Checklist

### Week 1-2: Foundations
- [ ] Core resource types mastered
- [ ] Can write deployment manifests
- [ ] Understand rolling updates

### Week 2-3: Networking
- [ ] CNI plugins understood
- [ ] Service discovery clear
- [ ] NetworkPolicies practical

### Week 3-4: Architecture
- [ ] EKS setup understood
- [ ] AKS features known
- [ ] Bare metal challenges identified

### Week 4-5: Observability
- [ ] Prometheus queries work
- [ ] Logging stack deployed
- [ ] Tracing understood

### Week 6-7: Cloud & Multi-Cloud
- [ ] AWS tools compared
- [ ] Azure tools compared
- [ ] Vault multi-cloud strategy

### Week 7-9: Hands-On
- [ ] 3 practicals complete
- [ ] 6 practicals complete
- [ ] 10 practicals complete

### Week 10: Interview Ready
- [ ] 3 mock interviews done
- [ ] Question bank reviewed
- [ ] Design patterns practiced

---

## Success Criteria

**Interview Ready When:**
- Can answer all 26 interview questions
- Can complete 10 practicals without help
- Can design multi-cloud architecture
- Can troubleshoot real scenarios
- Score 80%+ on mock interviews

