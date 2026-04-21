# Kubernetes Interview Preparation - Complete Study Package
## Master Index & Navigation Guide

---

## 📚 Study materials (folder: `/home/frontier/devops-interview-prep/aws-devops/kuberenets`)

Paths below are **filenames in that folder** (same directory as this index).

### 1. **CRDs & Operators Deep Dive** 
📄 **File:** `crds_operators_guide.md` (~900 lines)

**Topics Covered:**
- CRD structure, versioning, validation schemas
- OpenAPI v3 validation, CEL expressions
- Status subresources and finalizers
- Validating & mutating webhooks
- Operator pattern & reconciliation loop
- Operator maturity levels
- Workspace examples (Jaeger, cert-manager, ALB controller)
- 5 detailed interview questions with answers
- Hands-on: Build a ConfigWatcher operator

**Key Takeaway:** CRD = API Extension, Operator = Automation + Domain Knowledge

---

### 2. **AWS EKS Architecture Deep Dive**
📄 **File:** `eks_architecture_guide.md` (~1,100 lines)

**Topics Covered:**
- EKS control plane (multi-AZ, managed, 99.95% SLA)
- Node groups (managed, self-managed, Fargate)
- IRSA: IAM Roles for Service Accounts (step-by-step setup)
- Managed add-ons (VPC CNI, kube-proxy, CoreDNS, EBS CSI)
- VPC integration, security groups per pod, secondary CIDR
- Production EKS checklist
- Design scenarios & troubleshooting
- Workspace reference: eksv2/ Terraform configuration

**Key Takeaway:** IRSA is critical for pod-to-AWS permissions (not credentials in Secret)

---

### 3. **Azure AKS Architecture Deep Dive**
📄 **File:** `aks_architecture_guide.md` (~950 lines)

**Topics Covered:**
- AKS control plane (managed by Azure, separate tenant)
- Node pools (system mandatory, user optional)
- Managed identity vs service principal (legacy)
- Workload identity federation (pod-level identity)
- Azure CNI vs Kubenet comparison
- Azure AD RBAC (not K8s RBAC)
- Virtual nodes (serverless containers)
- AKS vs EKS: Complete comparison matrix
- Multi-cloud scenarios

**Key Takeaway:** Managed identity replaces service principal (like IRSA but Azure-specific)

---

### 4. **Bare Metal Kubernetes**
📄 **File:** `bare_metal_kubernetes_guide.md` (~800 lines)

**Topics Covered:**
- kubeadm cluster bootstrap (step-by-step)
- CNI choice for on-prem (Calico vs Cilium)
- HA control plane (3+ masters, etcd cluster, load balancer)
- Storage options (local, NFS, Ceph distributed)
- Networking without cloud provider
- External load balancer (MetalLB, HAProxy)
- On-prem-specific challenges & solutions
- On-prem vs cloud comparison

**Key Takeaway:** Calico preferred for on-prem (BGP routing, proven at scale)

---

### 5. **Remaining Modules Summary**
📄 **File:** `remaining_modules_summary.md` (~800 lines)

**Quick Reference for:**
- Multi-architecture deployments (x64, ARM64, buildx)
- Scaling & HA (HPA, cluster autoscaler, Karpenter, PDB, topology spread)
- Security deep dive (RBAC, Pod Security Standards, secrets, image scanning)
- Observability stack (Prometheus, Grafana, Loki, Jaeger, OpenTelemetry)
- Helm & GitOps (ArgoCD, templating, hooks)
- Hands-on practicals checklist (7 key labs)
- Workspace study guide (CPQ, CS, Terraform)
- Mock interview prep

**Key Takeaway:** Efficient 1-2 hour summaries for each topic

---

### 6. **Final Summary & Progress**
📄 **File:** `INTERVIEW_PREP_COMPLETE_FINAL.md` (~1,000 lines)

### 7. **Monitoring, Scanning, Cloud Tools**
📄 **File:** `MONITORING_SCANNING_CLOUD_TOOLS.md` — Prometheus, Grafana, Loki, ELK, Jaeger, Istio, Cilium, Trivy, Falco, AWS/Azure services, Vault.

### 8. **CI/CD (ArgoCD, Jenkins, GitHub Actions, Azure DevOps)**
📄 **File:** `CICD_DEPLOYMENT_TOOLS.md`

### 9. **Core resources & navigation**
📄 **Files:** `core_resources_interview_qa.md`, `core_resources_guide.yaml`, `README.md`, `study_summary.md`

**Includes:**
- Complete checklist of all topics mastered
- Interview preparation timeline (3-week plan)
- Success metrics (8-point checklist)
- Quick reference sheets (decision trees, PromQL snippets)
- Post-interview growth areas
- Resource summary & success metrics

---

## 🎯 How to Use This Study Package

### Week 1: Foundation Review (3-4 hours/day)
```
Day 1-2: Read CRDs & Operators guide (new knowledge foundation)
Day 3-4: Read EKS guide (depth on primary platform)
Day 5-6: Read AKS guide (understand alternatives)
Day 7: Read Bare Metal guide (appreciate on-prem complexity)
```

### Week 2: Practice & Comparison (3-4 hours/day)
```
Day 1-2: Workspace code review (Terraform, Helm)
Day 3-4: Design scenarios (startup EKS, enterprise AKS)
Day 5-6: Troubleshooting practice (5 critical paths)
Day 7: Multi-cloud architecture design
```

### Week 3: Interview Ready (4-5 hours/day)
```
Day 1-2: Mock interviews (colleague/mentor)
Day 3-4: Review weak areas (deep-dive specific topics)
Day 5: Final mock interview with scoring
Day 6: Rest & prepare mentally
Day 7: Interview day! 🚀
```

---

## 📋 Complete Topic Checklist

### Kubernetes Core (✅ Completed Earlier)
- [x] Pod lifecycle & QoS classes
- [x] Deployment, StatefulSet, DaemonSet, Job, CronJob
- [x] Services & service discovery
- [x] Ingress & load balancing
- [x] PersistentVolumes, PersistentVolumeClaims, StorageClass
- [x] ConfigMaps & Secrets

### Advanced Networking (✅ Completed Earlier)
- [x] CNI plugins (Amazon VPC CNI, Calico, Cilium, Flannel, Weave)
- [x] Kubernetes network model
- [x] Service discovery (CoreDNS, kube-proxy, endpoint slices)
- [x] NetworkPolicy (ingress, egress, cross-namespace)
- [x] Debugging network connectivity

### Custom Resources & Automation (✅ NEW MODULE)
- [x] CRD structure & OpenAPI validation
- [x] Webhooks (validating & mutating)
- [x] Finalizers & custom cleanup
- [x] Operator pattern & reconciliation loops
- [x] Workspace operators (Jaeger, cert-manager, ALB)

### Platform Architecture (✅ NEW MODULES)
- [x] EKS (control plane, node groups, IRSA, add-ons, VPC)
- [x] AKS (control plane, node pools, managed identity, Azure AD)
- [x] Bare Metal (kubeadm, CNI, HA, storage)
- [x] Multi-cloud (design patterns, tooling)

### Advanced Topics (✅ Summary)
- [x] Multi-arch deployments (x64, ARM64)
- [x] Scaling (HPA, cluster autoscaler, Karpenter)
- [x] High availability (multi-AZ, PDB, topology spread)
- [x] Security (RBAC, PSS, image scanning, network policies)
- [x] Observability (Prometheus, Grafana, Loki, Jaeger)
- [x] Package management (Helm, templating)
- [x] Deployment (GitOps, ArgoCD)

---

## 🔍 Interview Question Bank (Ready)

### Design Questions (Can answer 15+ scenarios)
- Design production EKS cluster (any budget/scale)
- Design enterprise AKS cluster (compliance-heavy)
- Design bare-metal cluster (on-prem 500 nodes)
- Design multi-cloud (AWS + Azure + on-prem)
- Design for high-availability (zero downtime)
- Design security architecture (zero-trust)
- Design observability platform
- Design Helm deployment pipeline

### Troubleshooting Scenarios (Can solve 10+ paths)
- Pod pending / CrashLoopBackOff
- Service unreachable (cross-namespace)
- Pod can't access S3 (IRSA debugging)
- Pod can't access Key Vault (managed identity)
- High latency (performance debugging)
- Network policy blocking traffic
- Failed deployment (Helm chart issues)
- Cluster performance degradation
- Breach response (audit trail)

### Technical Deep-Dives (Can explain 10+ concepts)
- IRSA vs managed identity vs service principal
- EKS vs AKS vs bare-metal trade-offs
- Operator pattern & reconciliation loop
- HPA vs cluster autoscaler vs Karpenter
- CNI options & when to use each
- Prometheus PromQL queries
- Distributed tracing & correlation
- Helm templating & dependency management
- GitOps & ArgoCD sync policies
- Vault dynamic secrets & policies

---

## 🎓 Success Metrics (You're Ready When...)

✅ Can design production cluster in 10 minutes (any platform)
✅ Can troubleshoot pod-to-S3 access end-to-end in 5 minutes  
✅ Can compare EKS vs AKS vs bare-metal with trade-off analysis
✅ Can write YAML manifests from scratch (Deployment, StatefulSet, NetworkPolicy)
✅ Can explain operator pattern with real-world example
✅ Can design observability platform (metrics + logs + traces)
✅ Can discuss trade-offs thoughtfully (cost, complexity, security)
✅ Can cite workspace examples in answers

---

## 📖 Quick Navigation

| Topic | File | Lines | Time |
|-------|------|-------|------|
| CRDs & Operators | crds_operators_guide.md | 900 | 2 hrs |
| EKS Architecture | eks_architecture_guide.md | 1,100 | 2.5 hrs |
| AKS Architecture | aks_architecture_guide.md | 950 | 2 hrs |
| Bare Metal K8s | bare_metal_kubernetes_guide.md | 800 | 2 hrs |
| Advanced Topics | remaining_modules_summary.md | 800 | 2 hrs |
| Monitoring & cloud tools | MONITORING_SCANNING_CLOUD_TOOLS.md | — | 2–3 hrs |
| CI/CD | CICD_DEPLOYMENT_TOOLS.md | — | 2–3 hrs |
| Core YAML + Q&A | core_resources_guide.yaml, core_resources_interview_qa.md | — | 1–2 hrs |
| **TOTAL** | **See README.md** | **10,000+** | **40–50 hrs** |

---

## 🚀 Ready to Interview!

**Study Material Status:** ✅ COMPLETE (4,500+ lines)
**Modules Covered:** ✅ ALL 16 (100% complete)  
**Interview Readiness:** 8-9/10 (with practice)
**Time to Complete:** 40-50 hours total (mix of reading + practice)

---

## Next Steps

1. **This Week:** Review guides section-by-section
2. **Next Week:** Practice scenarios & mock interviews  
3. **Before Interview:** One final review of weak areas
4. **Interview:** Use frameworks & cite real examples

---

**Generated:** 2026-04-21 (paths corrected for `/home/frontier/devops-interview-prep/aws-devops/kuberenets`)  
**Total Study Material:** 10,000+ lines (all guides combined)  
**Topics Mastered:** 50+  
**Interview Scenarios:** 25+  
**Code Examples:** 100+

**You've got this! 💪**
