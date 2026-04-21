# 06 — Operations, troubleshooting, and GitOps

## 1. kubectl essentials

```bash
kubectl config use-context my-cluster
kubectl get all -n my-ns
kubectl describe pod POD -n my-ns
kubectl logs POD -c CONTAINER --previous
kubectl exec -it POD -n my-ns -- sh
kubectl port-forward svc/web 8080:80 -n my-ns
kubectl get events -n my-ns --sort-by='.lastTimestamp'
```

**Dry run / diff:**

```bash
kubectl apply -f app.yaml --dry-run=client -o yaml
kubectl diff -f app.yaml
```

---

## 2. Labels and selectors

Labels index and group objects; **selectors** tie Deployments to Pods, Services to Pods.

```bash
kubectl get pods -l app=web
kubectl label pod my-pod tier=frontend --overwrite
```

**Annotations:** non-identifying metadata (build id, owner contact).

---

## 3. Namespaces and quotas

**ResourceQuota** caps total CPU/memory/object count per namespace. **LimitRange** sets defaults/limits per Pod in namespace.

---

## 4. Debugging workloads

| Symptom | Checks |
|---------|--------|
| **ImagePullBackOff** | Image name, pull secrets, registry auth |
| **CrashLoopBackOff** | `kubectl logs --previous`, command/args, probes |
| **Pending** | `kubectl describe pod` — scheduling, PVC binding, taints |
| **OOMKilled** | Raise memory limit or fix leak |

**Ephemeral debug container** (debug feature / 1.23+):

```bash
kubectl debug POD -it --image=busybox:1.36 --target=app
```

**Copy Pod for isolation:**

```bash
kubectl debug POD -it --copy-to=debug-copy --share-processes --container=app --image=busybox:1.36
```

---

## 5. Common failure modes

- **Service no endpoints** — label mismatch, readiness failing, wrong port
- **DNS failures** — CoreDNS pods, `ndots` in resolv.conf, NetworkPolicy
- **CNI** — node NotReady, IP pool exhaustion
- **etcd** — slow API, quorum loss (control plane)
- **Certificate expiry** — kubeadm clusters: check apiserver/kubelet certs

---

## 6. Helm basics

Package manager: **Chart** = templates + `values.yaml`.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-nginx bitnami/nginx -n apps --create-namespace
helm upgrade my-nginx bitnami/nginx -f custom-values.yaml
helm rollback my-nginx 1 -n apps
```

**Interview:** Helm vs Kustomize — templating + release history vs overlay patches.

---

## 7. GitOps (high level)

**Declarative** desired state in Git; controller reconciles cluster.

| Tool | Controller |
|------|------------|
| **Argo CD** | Application CRD → sync manifests / Helm |
| **Flux** | GitRepository + Kustomization / HelmRelease |

Benefits: audit trail, drift detection, promotion workflows. **Caveat:** secrets handling (SOPS, external secrets).

---

## 8. Upgrades and backups (interview depth)

- **Control plane:** rolling API servers; etcd backup/restore is critical (snapshots).
- **Worker nodes:** cordon/drain, replace AMIs, upgrade kubelet.
- **Workload compatibility:** check deprecated APIs (`kubectl convert` deprecated — use **pluto** or **kube-no-trouble**).

```bash
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
kubectl uncordon NODE
```

---

## 9. Observability touchpoints

- **Metrics:** metrics-server, Prometheus, **kubectl top**
- **Logs:** cluster logging agent (DaemonSet) → Loki/Elasticsearch/CloudWatch
- **Tracing:** OpenTelemetry sidecar / service mesh

*(Deep dive: parent repo Phase 4 + `kuberenets/MONITORING_SCANNING_CLOUD_TOOLS.md` if present.)*

---

## Quick recap

- **describe**, **events**, **logs --previous**, **debug** are your first tools.
- **Helm** packages releases; **GitOps** keeps cluster aligned with Git.
- **Drain/cordon** for node maintenance; **etcd** backup for disaster recovery.

### Interview prompts

1. How do you safely evict workloads from a node before patching?
2. What is the difference between Helm upgrade and `kubectl apply` for an app?
3. How would you find which object is hogging a namespace quota?
4. Name three causes of a Pod stuck in `Pending`.
