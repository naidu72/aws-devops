# 08 — Hands-on labs (CKA-style)

**Cluster:** kind, minikube, k3d, or any 1.26+ cluster. Adjust `StorageClass` names (`standard` vs `gp2`) for your environment.

**Convention:** Create a workspace namespace for each session: `kubectl create ns labs && kubectl config set-context --current --namespace=labs`

---

## Lab 1 — Multi-container Pod with shared volume

**Task:** Run a Pod `logger` with two containers: `generator` writes a timestamp every 2s to `/data/log.txt`; `tailer` runs `tail -f /data/log.txt`. Use `emptyDir` shared volume.

<details>
<summary>Solution outline</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: logger
spec:
  volumes:
    - name: data
      emptyDir: {}
  containers:
    - name: generator
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - while true; do date >> /data/log.txt; sleep 2; done
      volumeMounts:
        - name: data
          mountPath: /data
    - name: tailer
      image: busybox:1.36
      command: ["/bin/sh", "-c", "tail -f /data/log.txt"]
      volumeMounts:
        - name: data
          mountPath: /data
```

```bash
kubectl apply -f logger.yaml
kubectl logs logger -c tailer -f
```

</details>

---

## Lab 2 — Deployment rolling update and rollback

**Task:** Create Deployment `web` with `nginx:1.25-alpine`, 3 replicas. Record change-cause. Update image to `nginx:1.26-alpine`, watch rollout. Roll back to previous revision.

<details>
<summary>Solution outline</summary>

```bash
kubectl create deployment web --image=nginx:1.25-alpine --replicas=3 --record   # or use annotation
# modern: kubectl annotate deployment web kubernetes.io/change-cause="v1"
kubectl set image deployment/web nginx=nginx:1.26-alpine
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

</details>

---

## Lab 3 — Probes: fix a failing readiness

**Task:** Pod runs `nginx` but readiness probes `/healthz` which 404s. Add a **readinessProbe** on `/` or create `/healthz` via custom config. Verify Service endpoints.

<details>
<summary>Solution outline</summary>

Simplest fix — probe root:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 5
```

```bash
kubectl get endpoints web
```

</details>

---

## Lab 4 — Expose Deployment with ClusterIP and debug DNS

**Task:** Expose `web` Deployment on port 80 ClusterIP. Run a temporary `busybox` Pod and `wget -qO- http://web:80`.

<details>
<summary>Solution outline</summary>

```bash
kubectl expose deployment web --port=80 --target-port=80
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web
```

</details>

---

## Lab 5 — Ingress path-based routing (nginx ingress)

**Prereq:** Install [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/) or your cluster’s equivalent.

**Task:** Two Deployments `api-v1` and `api-v2`. One Ingress: `/v1` → v1 Service, `/v2` → v2 Service.

<details>
<summary>Solution outline</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 80
```

Use `kubectl port-forward` to ingress controller or hit LB hostname.

</details>

---

## Lab 6 — NetworkPolicy: deny all ingress except same-namespace

**Prereq:** CNI with NetworkPolicy support (Calico, Cilium, etc.).

**Task:** Namespace `backend` with Pod `api` (label `app=api`). Only allow TCP 8080 from namespace labeled `name=frontend`.

<details>
<summary>Solution outline</summary>

Label frontend namespace: `kubectl label ns frontend name=frontend`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend
      ports:
        - port: 8080
        protocol: TCP
```

**Note:** Default-deny-all often requires a second policy or CNI-specific behavior—confirm with your CNI docs.

</details>

---

## Lab 7 — ConfigMap volume mount

**Task:** Mount `app.conf` from ConfigMap as file at `/config/app.conf` in a `busybox` Pod; `cat` it.

<details>
<summary>Solution outline</summary>

```bash
kubectl create configmap app-cfg --from-literal=app.conf="log_level=debug"
```

```yaml
spec:
  volumes:
    - name: cfg
      configMap:
        name: app-cfg
  containers:
    - name: c
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: cfg
          mountPath: /config
```

</details>

---

## Lab 8 — PVC and Pod (dynamic provisioning)

**Task:** Create PVC 1Gi (your default StorageClass). Pod mounts PVC at `/data` and writes a file.

<details>
<summary>Solution outline</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c", "echo hello > /data/file && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
```

```bash
kubectl get pvc
kubectl exec writer -- cat /data/file
```

</details>

---

## Lab 9 — RBAC: read-only ServiceAccount

**Task:** In namespace `labs`, create SA `reader`. Grant **only** get/list/watch on Pods. Verify with a token (or `kubectl --as=system:serviceaccount:labs:reader` if impersonation allowed).

<details>
<summary>Solution outline</summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: reader
  namespace: labs
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-ro
  namespace: labs
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-ro-bind
  namespace: labs
subjects:
  - kind: ServiceAccount
    name: reader
    namespace: labs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-ro
```

```bash
kubectl auth can-i list pods --as=system:serviceaccount:labs:reader -n labs   # yes
kubectl auth can-i create pods --as=system:serviceaccount:labs:reader -n labs # no
```

</details>

---

## Lab 10 — Debug CrashLoopBackOff

**Task:** Given a broken Pod manifest (wrong command or bad image), use `describe`, `logs --previous`, and fix.

<details>
<summary>Solution outline</summary>

```bash
kubectl describe pod BROKEN
kubectl logs BROKEN --previous
# Fix: image tag, entrypoint, env, or probe thresholds
kubectl apply -f fixed.yaml
```

</details>

---

## Lab 11 — Cord and drain a node (simulation)

**Task:** If multi-node: `cordon` a node, `drain` with `--ignore-daemonsets --delete-emptydir-data`, then `uncordon`.

<details>
<summary>Solution outline</summary>

```bash
kubectl cordon NODE
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
# maintenance...
kubectl uncordon NODE
```

On single-node kind/minikube, discuss **PDB** blocking drain.

</details>

---

## Lab 12 — Job with completions

**Task:** Run a Job that completes 5 times successfully with parallelism 2 (busybox sleep).

<details>
<summary>Solution outline</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: work
          image: busybox:1.36
          command: ["/bin/sh", "-c", "sleep 5"]
```

```bash
kubectl get jobs
kubectl logs job/batch
```

</details>

---

## Lab 13 — Taints, tolerations, and pod anti-affinity

**Needs:** A **multi-node** cluster (e.g. kind with 3 workers). On single-node clusters, run through commands and read `Events`—expect anti-affinity to **not** spread beyond one node.

**Task A — Taint:** Pick a node `NODE`, add `workload=demo:NoSchedule`. Create a Pod **without** tolerations; confirm `Pending` and message about taint. Add toleration; confirm `Running`.

**Task B — Anti-affinity:** Deploy 3 replicas of `nginx` with **preferred** `podAntiAffinity` on `kubernetes.io/hostname`. `kubectl get pods -o wide` and observe spread when multiple nodes exist.

<details>
<summary>Solution outline</summary>

```bash
kubectl get nodes
kubectl taint nodes NODE workload=demo:NoSchedule
kubectl run blocked --image=nginx:1.25-alpine --restart=Never
kubectl describe pod blocked   # 0/X nodes available: ... taint
kubectl delete pod blocked
```

Pod **with** toleration (schedules onto `NODE` even while tainted):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: allowed
spec:
  tolerations:
    - key: workload
      operator: Equal
      value: demo
      effect: NoSchedule
  containers:
    - name: nginx
      image: nginx:1.25-alpine
```

Remove taint before Task B (spread demo should use **untainted** nodes):  
`kubectl taint nodes NODE workload=demo:NoSchedule-`

**Task B — anti-affinity only** (no tolerations):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-web
spec:
  replicas: 3
  selector:
    matchLabels: { app: spread-web }
  template:
    metadata:
      labels: { app: spread-web }
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels: { app: spread-web }
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.25-alpine
```

```bash
kubectl get pods -l app=spread-web -o wide
```

</details>

---

## Lab 14 — Discover CRDs and explain a custom Kind

**Task:** List CRDs in the cluster. Pick one (e.g. from a installed operator or `cert-manager.io`). Run `kubectl explain <resource>` for its `spec`. If you have a throwaway cluster, apply the minimal **Widget** CRD from [09-crds-and-operators.md](09-crds-and-operators.md) and create one instance; then `kubectl get widgets` and delete both.

**Safety:** Do not install random CRDs on production; use kind/minikube only.

<details>
<summary>Solution outline</summary>

```bash
kubectl get crd
kubectl api-resources --api-group=cert-manager.io   # example group if present
kubectl explain certificate.spec --api-version=cert-manager.io/v1   # adjust if installed
```

After applying a test CRD from ch.09:

```bash
kubectl apply -f crd-widget.yaml
kubectl apply -f widget-instance.yaml
kubectl get widgets
kubectl delete widget demo
kubectl delete crd widgets.stable.example.com
```

If deletion hangs, check `metadata.finalizers` on the CR or CRD.

</details>

---

## Suggested scoring (self-check)

| Skill | Labs |
|--------|------|
| Pod composition | 1 |
| Deployments | 2, 3 |
| Services/DNS | 4 |
| Ingress | 5 |
| Policy | 6 |
| Config/storage | 7, 8 |
| RBAC | 9 |
| Troubleshooting | 10 |
| Ops | 11 |
| Batch | 12 |
| Affinity / taints | 13 |
| CRDs | 14 |

Repeat any lab **blind** (no solution) until commands flow without notes.
