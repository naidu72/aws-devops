# Core Kubernetes resources — reference guide

**Folder:** `/home/frontier/devops-interview-prep/aws-devops/kuberenets`

| File | Use |
|------|-----|
| **`core_resources_guide.yaml`** | Apply with `kubectl apply -f core_resources_guide.yaml` (multi-document YAML). |
| **`core_resources_guide.md`** | This file — explanations + copy-paste snippets (same manifests as the `.yaml`). |
| **`core_resources_interview_qa.md`** | Interview Q&A and deeper scenarios. |

---

## Apply all resources at once

```bash
cd /home/frontier/devops-interview-prep/aws-devops/kuberenets
kubectl apply -f core_resources_guide.yaml
```

To remove the demo namespace and everything in it:

```bash
kubectl delete namespace demo --ignore-not-found
```

---

## Namespace

Creates a `demo` namespace used by the other examples below.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

---

## Pod

Single pod with CPU/memory requests and limits and HTTP health probes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  namespace: demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
```

---

## Deployment

Two replicas, rolling updates managed by the Deployment controller.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deploy
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
```

---

## Service (ClusterIP)

Stable virtual IP and DNS name for the Deployment pods (`demo-svc.demo.svc.cluster.local`).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 80
```

---

## Ingress

HTTP routing to the Service (requires an Ingress controller in the cluster).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ing
  namespace: demo
spec:
  rules:
  - host: demo.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-svc
            port:
              number: 80
```

---

## StatefulSet

Ordered pods with per-replica PVC via `volumeClaimTemplates` (needs a default StorageClass or adjust storage).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: demo-sts
  namespace: demo
spec:
  serviceName: demo-sts
  replicas: 1
  selector:
    matchLabels:
      app: demo-sts
  template:
    metadata:
      labels:
        app: demo-sts
    spec:
      containers:
      - name: app
        image: nginx:1.25
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

---

## DaemonSet

One pod per matching node (typical for node agents, log collectors).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: demo-ds
  namespace: demo
spec:
  selector:
    matchLabels:
      app: demo-ds
  template:
    metadata:
      labels:
        app: demo-ds
    spec:
      containers:
      - name: agent
        image: busybox:1.36
        command: ["sh", "-c", "while true; do sleep 3600; done"]
```

---

## Job

Runs a container to completion (`restartPolicy: Never`).

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: demo-job
  namespace: demo
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: run
        image: busybox:1.36
        command: ["sh", "-c", "echo ok"]
```

---

## CronJob

Runs on a schedule (cron syntax).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: demo-cron
  namespace: demo
spec:
  schedule: "0 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: run
            image: busybox:1.36
            command: ["sh", "-c", "date"]
```

---

## Sync note

The **`core_resources_guide.yaml`** file in this directory is the **canonical** multi-document manifest; it should stay in sync with the YAML blocks above when you edit either file.
