# PART 2: KUBERNETES STORAGE & VOLUME MANAGEMENT

## OVERVIEW: The Storage Stack

```
Application Pod
    ↓ (needs persistent data)
PVC (Persistent Volume Claim)
    ↓ (my app claims 10Gi storage, type: fast)
StorageClass (dynamic provisioner)
    ↓ (AWS gp3 volume with 3000 IOPS)
PV (Persistent Volume - actual storage)
    ↓ (bound to actual EBS volume vol-12345)
Physical Storage
    ↓ (actual disk on infrastructure)
```

---

## CORE CONCEPTS

### 1. PERSISTENT VOLUME (PV) - Cluster-Level Storage Resource

**Purpose:** Represent actual storage (EBS, NFS, GCS, etc.)

**Key Characteristics:**
- Cluster-level resource (not namespace-scoped)
- Created by admin or dynamic provisioner
- Has lifecycle independent of pods

**PV Example:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ebs-001
spec:
  capacity:
    storage: 10Gi  # How much storage
  
  accessModes:
    - ReadWriteOnce  # Only 1 pod can write at a time
  
  # Reclaim policy: what happens when PVC deleted?
  persistentVolumeReclaimPolicy: Retain
    # Options: Retain (keep), Delete (destroy), Recycle (format)
  
  # Which storage backend
  awsElasticBlockStore:
    volumeID: vol-12345
    fsType: ext4
```

**Access Modes:**
- **ReadWriteOnce (RWO):** Only 1 pod can write (EBS, GCE Disk) ← MOST COMMON
- **ReadOnlyMany (ROX):** Multiple pods read-only (NFS)
- **ReadWriteMany (RWX):** Multiple pods read/write (NFS, EFS, shared storage)

---

### 2. PERSISTENT VOLUME CLAIM (PVC) - Pod's Storage Request

**Purpose:** Pod says "I need 10Gi of fast storage"

**PVC Example:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  
  # Request specific amount
  resources:
    requests:
      storage: 10Gi
  
  # Which StorageClass to use
  storageClassName: fast-gp3
    # If omitted, uses cluster's default StorageClass
```

**Flow:**
1. Pod references PVC: `volumes: [name: postgres-pvc, persistentVolumeClaim: {claimName: postgres-pvc}]`
2. Kubernetes looks for matching PV
3. If found AND not already bound, bind PVC to PV
4. Pod gets access to storage

**Key Point:** PVC and PV are 1:1 (one PVC bound to one PV)

---

### 3. STORAGE CLASS - Dynamic Provisioning

**Purpose:** Automate PV creation when PVC requested

**Why it matters:**
- **Without StorageClass:** Admin creates PV manually (tedious, error-prone)
- **With StorageClass:** Pod requests storage, PV created automatically

**StorageClass Example (AWS):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-gp3
provisioner: ebs.csi.aws.com  # Which provisioner to use
parameters:
  type: gp3        # EBS volume type
  iops: "3000"     # Provisioned IOPS
  throughput: "125" # MB/s
  encrypted: "true"
  kms_key_id: "arn:aws:kms:..."
  
allowVolumeExpansion: true  # Can PVC grow? (yes)
volumeBindingMode: WaitForFirstConsumer  # Bind when pod scheduled (smarter)
reclaimPolicy: Delete  # Delete storage when PVC deleted
```

**Common Provisioners:**
| Cloud | Provisioner | Default | Features |
|-------|-------------|---------|----------|
| AWS | ebs.csi.aws.com | gp3 | IOPS tuning, encryption |
| Azure | disk.csi.azure.com | managed-csi | Multiple tiers (standard, premium) |
| GCP | pd.csi.storage.gke.io | standard | Performance tuning |
| On-Prem | kubernetes.io/no-provisioner | N/A | Manual PV creation |

---

## VOLUME TYPES

### 1. emptyDir - Ephemeral (LOST on pod deletion)

**Use case:** Temporary storage shared between containers in pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  containers:
  - name: writer
    image: alpine
    command: ['sh', '-c', 'while true; do echo $RANDOM > /data/random; sleep 1; done']
    volumeMounts:
    - name: shared
      mountPath: /data
  
  - name: reader
    image: alpine
    command: ['sh', '-c', 'while true; do cat /data/random; sleep 1; done']
    volumeMounts:
    - name: shared
      mountPath: /data
  
  volumes:
  - name: shared
    emptyDir: {}  # Empty directory, lost when pod deleted

# Data flow: writer → /data → reader (same pod)
```

**Storage:** Node's local disk (fast but temporary)
**Lifespan:** Pod's lifetime (deleted when pod deleted)
**Use case:** Logging, temporary processing, container communication

---

### 2. hostPath - Node's Filesystem (NOT Portable)

**Use case:** Access node's filesystem (monitoring agents, log collection)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monitor-node
spec:
  containers:
  - name: monitor
    image: prom/node-exporter
    volumeMounts:
    - name: rootfs
      mountPath: /rootfs
      readOnly: true
  
  volumes:
  - name: rootfs
    hostPath:
      path: /  # Mount node's root filesystem
      type: Directory  # Must exist
```

**Dangers:**
- Not portable (tied to specific node)
- Security risk (direct access to node filesystem)
- Can't use for persistent data across nodes

**Real use:** DaemonSets for monitoring, log collection

---

### 3. AWS EBS - Persistent Block Storage

**Use case:** Databases, stateful applications (EKS)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce  # EBS = single pod only
  
  awsElasticBlockStore:
    volumeID: vol-12345abcde  # Actual EBS volume
    fsType: ext4

---
# Modern way (StorageClass):
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi
```

**Important:**
- **Single AZ:** EBS volume bound to specific AZ → Pod can only run in that AZ
- **Failure scenario:** If node dies in AZ-a, pod can't reschedule to AZ-b (volume not available)
- **Multi-AZ HA:** Use RDS (managed database) instead, or EFS (shared filesystem)

---

### 4. AWS EFS - Shared Filesystem (ReadWriteMany)

**Use case:** Multiple pods reading/writing same data across AZs

```yaml
# First: EFS CSI driver must be installed
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  basePath: "/"
  subPathPattern: "${.PVC.namespace}/${.PVC.name}"

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany  # Multiple pods!
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi
```

**Why EFS over EBS:**
- Multi-AZ: Data available across all AZs
- ReadWriteMany: Multiple pods can write simultaneously
- Multi-pod scaling: Scale horizontally across AZs

**Trade-off:** EFS slower than EBS (network filesystem vs block device)

---

### 5. Azure Disk - Persistent Block Storage (AKS)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_LRS  # Premium, Standard, UltraSSD
  cachingmode: ReadOnly
  maxShares: "3"  # Multiple pods with UltraSSD

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-disk-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 50Gi
```

---

### 6. Azure Files - Shared Filesystem (AKS)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile
provisioner: file.csi.azure.com
parameters:
  skuName: Standard_LRS  # Standard or Premium
  storageAccount: mystorageaccount

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-files-pvc
spec:
  accessModes:
    - ReadWriteMany  # Multiple pods!
  storageClassName: azurefile
  resources:
    requests:
      storage: 100Gi
```

---

### 7. ConfigMap Volume - Config Data (Read-Only)

**Use case:** Inject config files into pod

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.yaml: |
    database:
      host: postgres.databases.svc.cluster.local
      port: 5432
    cache:
      ttl: 3600

---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  
  volumes:
  - name: config
    configMap:
      name: app-config
      # Files appear in /etc/config/: app.yaml
```

**Features:**
- Read-only (prevents accidental modification)
- Hot-reload: Changes to ConfigMap don't auto-update pod (pod restart needed)
- Size limit: ~1MB per ConfigMap

---

### 8. Secret Volume - Sensitive Data (Encrypted)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=  # base64: admin
  password: cGFzc3dvcmQxMjM=  # base64: password123

---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # Expose as environment variables
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    
    volumeMounts:
    # OR mount as files
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  
  volumes:
  - name: secrets
    secret:
      secretName: db-credentials
      # Files: /etc/secrets/username, /etc/secrets/password
```

**Security Note:**
- Base64 encoded (not encrypted!)
- For real security: Use external secret vault (Vault, AWS Secrets Manager)
- Encrypt secrets at rest in etcd

---

### 9. downwardAPI - Pod Metadata as Files

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api
  labels:
    app: myapp
    env: prod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: "pod-name"
        fieldRef:
          fieldPath: metadata.name
      - path: "pod-namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "requests-cpu"
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
```

**Use case:** Pod discovering its own metadata programmatically

---

## StatefulSet + PVC INTEGRATION

### Real Example: PostgreSQL Cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres  # Headless service
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
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
        image: postgres:13
        ports:
        - containerPort: 5432
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  # AUTOMATIC PVC creation per replica
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 50Gi

# Result:
# - postgres-0 pod + data-postgres-0 PVC + vol-xxxxx1 EBS
# - postgres-1 pod + data-postgres-1 PVC + vol-xxxxx2 EBS
# - postgres-2 pod + data-postgres-2 PVC + vol-xxxxx3 EBS
```

**Flow:**
1. StatefulSet creates postgres-0 pod
2. Kubernetes sees volumeClaimTemplates
3. Creates PVC named data-postgres-0
4. StorageClass gp3 provisions EBS volume
5. PVC bound to EBS volume
6. Pod mounts volume at /var/lib/postgresql/data
7. Repeat for postgres-1, postgres-2

---

## STORAGE CLASS BEST PRACTICES

### For Development:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-dev
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  iops: "100"
reclaimPolicy: Delete  # Clean up when done
allowVolumeExpansion: true
```

### For Production:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-prod
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kms_key_id: "arn:aws:kms:..."
volumeBindingMode: WaitForFirstConsumer  # Bind on scheduling
reclaimPolicy: Retain  # Keep data if PVC deleted
allowVolumeExpansion: true
```

---

## INTERVIEW QUESTIONS

**Q: "A pod lost data after restart. Why?"**
A: If using emptyDir volume, data is ephemeral (lost on pod deletion).
   Solution: Use PVC with persistent StorageClass (EBS, EFS, etc.)

**Q: "Can I use EBS for multi-pod shared storage?"**
A: No, EBS is block storage (ReadWriteOnce only).
   Solution: Use EFS (ReadWriteMany, network filesystem).

**Q: "How do I expand a PVC?"**
A: 
```bash
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
# StorageClass must allow: allowVolumeExpansion: true
```

**Q: "What's the difference between PV and PVC?"**
A: PV = physical storage (admin creates), PVC = storage request (pod creates/uses).
   StorageClass = automation layer (provisions PV when PVC requested).

---

