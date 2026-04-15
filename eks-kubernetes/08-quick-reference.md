# 08 - Quick Reference: EKS and Kubernetes Cheat Sheet

## kubectl Commands Cheat Sheet

### Cluster Info

```bash
kubectl cluster-info
kubectl get nodes
kubectl describe node <node>
kubectl get nodes -o wide
kubectl top nodes
kubectl top pods -A
```

### Namespaces

```bash
kubectl create namespace production
kubectl get namespaces
kubectl delete namespace dev
kubectl get pods -n production
kubectl get all -n production

# Set default namespace
kubectl config set-context --current --namespace=production
```

### Pods

```bash
kubectl get pods -n production
kubectl get pods -n production -o wide
kubectl describe pod <pod> -n production
kubectl logs <pod> -n production
kubectl logs <pod> -n production --previous
kubectl exec -it <pod> -n production -- bash
kubectl delete pod <pod> -n production
kubectl port-forward pod/<pod> 8080:8080 -n production
```

### Deployments

```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments -n production
kubectl describe deployment <deployment> -n production
kubectl scale deployment <deployment> --replicas=5 -n production
kubectl set image deployment/<deployment> <container>=<image>:<tag>
kubectl rollout status deployment/<deployment> -n production
kubectl rollout history deployment/<deployment> -n production
kubectl rollout undo deployment/<deployment> -n production
kubectl set env deployment/<deployment> LOG_LEVEL=INFO
kubectl patch deployment <deployment> -p '{"spec":{"replicas":3}}'
```

### Services

```bash
kubectl get services -n production
kubectl describe svc <service> -n production
kubectl get endpoints <service> -n production
kubectl port-forward svc/<service> 8080:80 -n production
```

### ConfigMaps & Secrets

```bash
# ConfigMaps
kubectl create configmap app-config --from-literal=LOG_LEVEL=INFO
kubectl get configmaps -n production
kubectl describe cm <configmap> -n production
kubectl edit cm <configmap> -n production

# Secrets
kubectl create secret generic db-creds --from-literal=password=secret123
kubectl get secrets -n production
kubectl describe secret <secret> -n production
```

### RBAC

```bash
kubectl get roles -n production
kubectl get rolebindings -n production
kubectl describe role <role> -n production
kubectl create rolebinding <name> --clusterrole=view --serviceaccount=production:default
kubectl auth can-i get pods -n production --as=system:serviceaccount:production:default
```

### Network Policies

```bash
kubectl get networkpolicies -n production
kubectl describe networkpolicy <policy> -n production
```

### Labels & Selectors

```bash
kubectl get pods -n production -l app=payment-service
kubectl get pods -n production -l app=payment-service,version=v1
kubectl label pods <pod> app=payment-service -n production
kubectl label pods <pod> app=payment-service --overwrite -n production
```

### Resource Management

```bash
kubectl top nodes
kubectl top pods -n production
kubectl describe node <node>
kubectl get resourcequotas -n production
```

---

## Common YAML Templates

### Deployment Template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-name
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-name
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: app-name
        version: v1
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - app-name
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: registry/app:v1.0
        ports:
        - containerPort: 8080
        env:
        - name: LOG_LEVEL
          value: "INFO"
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service Template

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
  namespace: production
spec:
  type: ClusterIP  # or LoadBalancer, NodePort
  selector:
    app: app-name
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```

### Ingress Template

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
```

### StatefulSet Template

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ebs-gp3
      resources:
        requests:
          storage: 100Gi
```

### Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: app-name
```

### Network Policy (Deny All)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
```

---

## Istio Commands

```bash
# Install Istio
istioctl install --set profile=production -y

# Sidecar injection
kubectl label namespace production istio-injection=enabled

# Verify installation
istioctl verify-install

# Check virtual services
kubectl get virtualservices -n production
kubectl describe vs <name> -n production

# Check destination rules
kubectl get destinationrules -n production
kubectl describe dr <name> -n production

# View traffic flow
istioctl analyze -n production

# Debug proxy
kubectl exec -it <pod> -n production -c istio-proxy -- bash
  # Inside: curl localhost:15000/stats/prometheus (Envoy stats)
```

---

## Helm Commands

```bash
# Repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo <chart>

# Install
helm install <release> <chart> -n <namespace> -f values.yaml

# Upgrade
helm upgrade <release> <chart> -n <namespace> -f values.yaml

# Rollback
helm rollback <release> -n <namespace>

# History
helm history <release> -n <namespace>

# List
helm list -n <namespace>

# Get values
helm get values <release> -n <namespace>

# Template (dry-run)
helm template <release> <chart> -f values.yaml

# Lint
helm lint <chart>

# Delete
helm delete <release> -n <namespace>
```

---

## Troubleshooting Checklist

### Pod Not Running

- [ ] Check pod events: `kubectl describe pod <name>`
- [ ] Check pod logs: `kubectl logs <pod>`
- [ ] Check resource requests: Is node capacity sufficient?
- [ ] Check image: Does image exist in registry?
- [ ] Check image pull secrets: Can pod pull the image?

### Service Not Accessible

- [ ] Service exists: `kubectl get svc`
- [ ] Service has endpoints: `kubectl get endpoints`
- [ ] Pods match selector: `kubectl get pods -l <selector>`
- [ ] Port mapping correct: `kubectl describe svc`
- [ ] Network policies blocking: `kubectl get networkpolicies`

### Deployment Not Scaling

- [ ] Replicas set: `kubectl describe deployment`
- [ ] Pods pending: `kubectl get pods`
- [ ] Node capacity: `kubectl describe node`
- [ ] Resource requests: Are they realistic?

### DNS Issues

- [ ] CoreDNS running: `kubectl get pods -n kube-system`
- [ ] Service exists: `kubectl get svc`
- [ ] Test from pod: `kubectl exec -it <pod> -- nslookup <service>`
- [ ] Check resolv.conf: `kubectl exec -it <pod> -- cat /etc/resolv.conf`

---

## Top 10 Most Important Facts

1. **Namespaces** provide logical isolation but NOT security isolation
2. **Requests** are used for scheduling; **limits** prevent pod from exceeding resources
3. **QoS Classes** determine eviction order: Guaranteed > Burstable > BestEffort
4. **Pod Disruption Budgets** prevent all replicas from being removed during maintenance
5. **Liveness probe** restarts pod; **readiness probe** removes from service
6. **Multi-AZ deployment** with 3+ replicas provides high availability
7. **StatefulSets** guarantee pod identity and persistent storage ordering
8. **DaemonSets** run exactly one pod per node
9. **Istio** provides traffic management, security, and observability at sidecar level
10. **Labels and selectors** are fundamental to Kubernetes resource management

---

## Quick Diagnosis Commands

```bash
# Everything in production namespace
kubectl get all -n production

# Specific resource status
kubectl get <resource> -n <namespace>

# Detailed diagnosis
kubectl describe <resource> <name> -n <namespace>

# Logs
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> -c <container>
kubectl logs deployment/<deployment> -n <namespace>

# Execute command in pod
kubectl exec -it <pod> -n <namespace> -- <command>

# Port forward for testing
kubectl port-forward pod/<pod> 8080:8080 -n <namespace>

# Show resource usage
kubectl top nodes
kubectl top pod -n <namespace>

# Edit resource live
kubectl edit <resource> <name> -n <namespace>
```

---

## Interview Quick Answers

**Q: "Describe your setup"**
A: "Multi-cluster EKS across DEV, QA, Staging, Production. Each has 1-3 nodes in DEV, 2-3 in QA/Staging, 6-12 in Production across 3 AZs. All use VPC CNI for pod networking. Istio for traffic management, Helm for deployments, Prometheus for monitoring."

**Q: "What's your deployment strategy?"**
A: "Rolling updates with maxSurge=1, maxUnavailable=0 for zero-downtime. For critical changes, canary deployments via Istio: 10% → 25% → 50% → 75% → 100% with 1-hour monitoring at each stage."

**Q: "How do you handle failures?"**
A: "Pod replicas spread across 3 AZs. If a pod crashes, Kubernetes restarts it. If a node fails, pods are evicted and rescheduled on healthy nodes. Pod Disruption Budgets maintain availability during maintenance. Istio outlier detection removes unhealthy pods automatically."

**Q: "How do you ensure security?"**
A: "RBAC limits permissions, network policies restrict traffic, no privileged containers, run as non-root, enable audit logging, scan images for vulnerabilities, mTLS between services via Istio."

**Q: "What's your monitoring strategy?"**
A: "Prometheus for Kubernetes metrics, CloudWatch for AWS metrics, OpenTelemetry for distributed tracing, CloudWatch Logs for aggregated logs. SLOs define acceptable availability (99.95%) and latency (<500ms)."

