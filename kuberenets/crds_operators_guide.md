# Custom Resource Definitions (CRDs) & Operators - Deep Dive

## Module Overview

**Learning Objectives:**
- Understand CRD structure and API conventions
- Master the operator pattern and reconciliation loop
- Review workspace operator examples
- Design custom resources for domain problems
- Interview scenarios and troubleshooting

**Estimated Study Time:** 8-10 hours
**Key Topics:** CRD structure, validation schemas, webhooks, status subresources, operator pattern, reconciliation loops

---

## Part 1: Custom Resource Definitions (CRDs) - Fundamentals

### 1.1 What is a CRD?

A **Custom Resource Definition** extends the Kubernetes API to support custom resources (beyond Pod, Service, etc.).

**Example: Without CRD**
```bash
# Kubernetes doesn't understand this
kubectl apply -f - <<EOF
apiVersion: mycompany.com/v1
kind: Database
metadata:
  name: mydb
spec:
  engine: postgresql
  version: "13"
  replicas: 3
EOF
# Error: the server doesn't have a resource type "databases"
```

**With CRD:**
1. First, define the CRD (schema, validation)
2. Then, Kubernetes accepts custom resources
3. Controllers (operators) act on custom resources

### 1.2 CRD Structure

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com  # Must be <plural>.<group>
spec:
  group: mycompany.com  # API group
  names:
    kind: Database      # Singular, CamelCase
    plural: databases   # Plural, lowercase
    shortNames:         # Optional abbreviations
    - db
  scope: Namespaced     # Namespaced or Cluster
  versions:
  - name: v1
    served: true        # Is this version served?
    storage: true       # Is this the storage version?
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - engine
            properties:
              engine:
                type: string
                enum: ["postgresql", "mysql", "mongodb"]
              version:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 10
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Pending", "Running", "Failed"]
              ready:
                type: boolean
              lastUpdateTime:
                type: string
                format: date-time
```

**Key Fields Explained:**

| Field | Purpose |
|-------|---------|
| `group` | API group (e.g., `mycompany.com`, `apiextensions.k8s.io`) |
| `kind` | Singular resource name (e.g., `Database`) |
| `plural` | Plural for API endpoints (e.g., `/apis/mycompany.com/v1/databases`) |
| `scope` | `Namespaced` (belongs to namespace) or `Cluster` (cluster-wide) |
| `versions` | Multiple API versions supported (for backward compatibility) |
| `schema` | OpenAPI 3.0 validation schema (type safety) |

### 1.3 Creating a CRD

```bash
# 1. Create the CRD
kubectl apply -f database-crd.yaml

# 2. Verify CRD created
kubectl get crd
kubectl get crd databases.mycompany.com -o yaml

# 3. Now you can create custom resources
kubectl apply -f - <<EOF
apiVersion: mycompany.com/v1
kind: Database
metadata:
  name: prod-db
  namespace: default
spec:
  engine: postgresql
  version: "13"
  replicas: 3
EOF

# 4. Query custom resources
kubectl get databases
kubectl get db  # Short name
kubectl get databases.mycompany.com
kubectl describe database prod-db
kubectl get database prod-db -o yaml
```

### 1.4 CRD Versioning & Migration

**Single Version (Simple):**
```yaml
versions:
- name: v1
  served: true
  storage: true
```

**Multiple Versions (With Migration):**
```yaml
versions:
- name: v1
  served: true
  storage: false  # Old version, not stored
  deprecated: true
  deprecationWarning: "v1 is deprecated, use v1beta1"
  
- name: v1beta1
  served: true
  storage: true   # Current storage version
  
- name: v1alpha1
  served: false   # No longer served
```

**Conversion:** If schema changes between versions, define conversion webhook:
```yaml
conversion:
  strategy: Webhook
  webhook:
    clientConfig:
      service:
        name: conversion-webhook
        namespace: default
        path: "/convert"
      port: 8443
```

---

## Part 2: CRD Advanced Features

### 2.1 Validation Schemas (OpenAPI v3)

**Basic validation:**
```yaml
spec:
  replicas:
    type: integer
    minimum: 1
    maximum: 10
  engine:
    type: string
    enum: ["postgresql", "mysql"]
```

**Complex validation:**
```yaml
spec:
  type: object
  required:
  - engine
  properties:
    engine:
      type: string
    backup:
      type: object
      properties:
        enabled:
          type: boolean
        schedule:
          type: string
          pattern: "^\\d{2}:\\d{2}$"  # HH:MM format
```

**Validation Rules (CEL expressions):**
```yaml
validationRules:
- rule: "self.spec.replicas <= 10"
  message: "replicas cannot exceed 10"
- rule: "self.spec.version != null"
  message: "version is required"
- rule: "self.spec.replicas > 0 if self.spec.engine == 'postgresql'"
  message: "PostgreSQL requires at least 1 replica"
```

### 2.2 Status Subresources

Separates desired state (spec) from observed state (status):

```yaml
versions:
- name: v1
  served: true
  storage: true
  schema:
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          # spec schema
        status:
          type: object
          # status schema
  subresources:
    status: {}  # Enable /status subresource
```

**Usage:**
```bash
# Update spec (desired state)
kubectl patch database prod-db --type merge -p '{"spec":{"replicas":5}}'

# Controller updates status (observed state)
kubectl patch database prod-db --subresource status --type merge -p '{"status":{"phase":"Running"}}'

# Get status
kubectl get database prod-db -o jsonpath='{.status.phase}'
```

**Why Separate?**
- Prevents accidental status overwrites when updating spec
- RBAC can grant different permissions (users update spec, controllers update status)
- API semantics: `kubectl apply` only touches spec

### 2.3 Finalizers (Custom Cleanup)

Finalizers prevent deletion until custom cleanup logic completes:

```yaml
apiVersion: mycompany.com/v1
kind: Database
metadata:
  name: prod-db
  finalizers:
  - mycompany.com/database-protection
spec:
  engine: postgresql
```

**Finalizer Logic (in Controller):**
```python
# Pseudo-code for controller logic
def reconcile(database):
    if database.metadata.deletionTimestamp is not None:
        # Object is being deleted
        if "mycompany.com/database-protection" in database.metadata.finalizers:
            # Run cleanup logic
            backup_to_s3(database)  # Backup data before deletion
            delete_rds_instance(database)  # Clean up resources
            
            # Remove finalizer
            database.metadata.finalizers.remove("mycompany.com/database-protection")
            update(database)
        return
    
    # Normal reconciliation (creation/update)
    ensure_database_exists(database)
    update_status(database)
```

**Deletion Flow:**
```
1. User: kubectl delete database prod-db
2. K8s marks object with deletionTimestamp
3. Controller sees deletionTimestamp
4. Controller runs cleanup (backup, teardown)
5. Controller removes finalizer
6. K8s deletes object
```

**Without Finalizers:**
- Database deleted immediately
- Data lost, resources not cleaned up

### 2.4 Webhooks (Admission Controllers)

**Two Types:**

#### Validating Webhooks (Read-Only)
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: database-validation
webhooks:
- name: validate.mycompany.com
  clientConfig:
    service:
      name: validation-webhook
      namespace: default
      path: "/validate"
    caBundle: LS0tLS1CRUdJTi... (base64 cert)
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["mycompany.com"]
    apiVersions: ["v1"]
    resources: ["databases"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
```

**Webhook Handler (validate):**
```python
def validate(admissionReview):
    database = admissionReview.request.object
    
    # Validation logic
    if database.spec.replicas > database.spec.resources.maxReplicas:
        return {
            "allowed": False,
            "message": "replicas cannot exceed maxReplicas"
        }
    
    return {"allowed": True}
```

#### Mutating Webhooks (Modify)
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: database-mutation
webhooks:
- name: mutate.mycompany.com
  clientConfig:
    service:
      name: mutation-webhook
      namespace: default
      path: "/mutate"
    caBundle: LS0tLS1CRUdJTi...
  rules:
  - operations: ["CREATE"]
    apiGroups: ["mycompany.com"]
    apiVersions: ["v1"]
    resources: ["databases"]
```

**Webhook Handler (mutate):**
```python
def mutate(admissionReview):
    database = admissionReview.request.object
    
    # Store original object for patch comparison
    import copy
    original = copy.deepcopy(database)
    
    # Set default values
    if database.spec.replicas is None:
        database.spec.replicas = 1
    
    if database.spec.backup is None:
        database.spec.backup = {"enabled": True, "schedule": "02:00"}
    
    # Return patch (only if modifications were made)
    patch = jsonpatch.make_patch(original, database)
    
    return {
        "allowed": True,
        "patch": base64.b64encode(json.dumps(patch).encode()).decode() if patch else ""
    }
```

---

## Part 3: Operator Pattern

### 3.1 What is an Operator?

An **Operator** = **CRD** + **Controller** + **Domain Knowledge**

**Pattern:**
```
CRD (Database resource spec)
  ↓
Controller (watches for changes)
  ↓
Reconciliation Loop (observe current state, act to reach desired state)
  ↓
External System (AWS RDS, on-prem database, etc.)
```

**Example Flow:**
```yaml
1. User creates:
   apiVersion: mycompany.com/v1
   kind: Database
   spec:
     engine: postgresql
     replicas: 3

2. Operator watches for new Database resources

3. Operator sees: "Database 'mydb' requested"

4. Operator checks: Is RDS instance running? No.

5. Operator acts: Create RDS instance with 3 read replicas

6. Operator updates status:
   status:
     phase: Running
     endpoint: mydb.c123.us-east-1.rds.amazonaws.com
```

### 3.2 Reconciliation Loop (Core Pattern)

**Pseudocode:**
```python
while True:
    for resource in watch(Database):
        try:
            # Step 1: Get current state
            current_state = get_external_state(resource)
            
            # Step 2: Compare with desired state
            desired_state = resource.spec
            
            # Step 3: Identify differences
            if current_state != desired_state:
                # Step 4: Take corrective action
                update_external_state(resource, desired_state)
                
            # Step 5: Update status with observed state
            update_status(resource, current_state)
            
        except Exception as e:
            # Requeue for retry
            requeue(resource, backoff=5s)
```

**Real Example: PostgreSQL Operator**
```python
def reconcile_database(database):
    # 1. Get current RDS instance
    current = describe_rds_instance(database.name)
    
    # 2. Check desired state
    desired_replicas = database.spec.replicas
    
    # 3. Fix discrepancies
    if current.replicas < desired_replicas:
        create_rds_read_replica(database.name)
    elif current.replicas > desired_replicas:
        delete_rds_read_replica(database.name)
    
    if current.version != database.spec.version:
        upgrade_rds_version(database.name, database.spec.version)
    
    # 4. Update status
    database.status.phase = "Running"
    database.status.endpoint = current.endpoint
    database.status.replicas = current.replicas
    update_status(database)
```

### 3.3 Key Characteristics

| Aspect | Details |
|--------|---------|
| **Continuous Reconciliation** | Constantly watches for drift from desired state |
| **Idempotent** | Calling reconcile multiple times = same result |
| **Self-Healing** | Auto-corrects if external state changes unexpectedly |
| **Level-Triggered** | Reacts to state, not events (if pod deleted, recreated) |
| **Exponential Backoff** | Retries on failure with increasing delay |

---

## Part 4: Operator Maturity Model

### Level 1: Basic Install
- CRD for resource definition
- Controller deploys application from manifest
- Example: Simple web server operator

### Level 2: Seamless Upgrades
- Support version upgrades
- Rolling updates with zero downtime
- Backup/restore capability
- Example: Database operator with schema migration

### Level 3: Full Lifecycle
- Day 2 operations automation
- Scaling, backup, disaster recovery
- Monitoring/alerting integration
- Example: Enterprise database operator

### Level 4: Deep Insights
- Metrics and observability
- Performance tuning recommendations
- Auto-scaling based on metrics
- Example: Advanced AI/ML platform operator

### Level 5: Auto-Pilot
- ML-driven optimization
- Self-tuning, self-healing
- Predictive scaling
- Example: Future enterprise platforms

---

## Part 5: Workspace Operator Examples

### 5.1 Jaeger Operator (Tracing)

**Location:** `hansen-cloud-native-suite/logging/modules/k8s/jaeger`

**What it does:**
- Users create `Jaeger` resource (CRD)
- Operator manages Jaeger deployment (agent, collector, UI)
- Handles storage backend (Elasticsearch) setup

**CRD Example:**
```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: observability
spec:
  strategy: production  # production or allinone
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
  ui:
    baseUrl: /jaeger
```

**Operator Flow:**
```
1. User creates Jaeger resource
2. Jaeger Operator observes it
3. Operator creates:
   - Agent DaemonSet
   - Collector Deployment
   - Query UI Service
   - Storage backend configuration
4. Status updated with UI endpoint
```

**Interview Question:** "How does Jaeger Operator simplify deployment compared to manual YAML manifests?"
**Answer:** 
- Single CRD resource instead of 5-10 manifests
- Operator handles defaults, validation
- Easy upgrades (change `image` in operator)
- Storage backend configuration automated

---

### 5.2 cert-manager Operator

**Location:** `hansen-cloud-native-suite/eksConfig/modules/k8s/certManager`

**CRDs Managed:**
- `Certificate` - Request TLS certificate
- `Issuer` / `ClusterIssuer` - Certificate authority
- `CertificateRequest` - Low-level cert request

**Example Usage:**
```yaml
# Step 1: Define Issuer (Let's Encrypt)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-key
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Step 2: Request certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - www.example.com
```

**Operator Flow:**
```
1. User creates Certificate resource
2. cert-manager operator observes it
3. Operator:
   - Validates domain ownership (HTTP-01 challenge)
   - Requests cert from Let's Encrypt
   - Stores cert in Secret (automatic)
4. Ingress uses Secret for TLS
5. Operator auto-renews 30 days before expiry
```

**Benefits:**
- No manual cert management
- Auto-renewal (never expires in production)
- Easy cert rotation

---

### 5.3 AWS Load Balancer Controller (Advanced)

**Location:** `hansen-cloud-native-suite/eksConfig/modules/k8s/loadBalancer`

**CRDs Managed:**
- `TargetGroupBinding` - Link K8s service to ALB/NLB target group
- `IngressClassParams` - Ingress-specific ALB parameters

**Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  annotations:
    alb.ingress.kubernetes.io/load-balancer-type: alb
spec:
  ingressClassName: alb
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Operator Flow:**
```
1. User creates Ingress
2. ALB Controller observes it
3. Controller:
   - Creates AWS ALB
   - Registers K8s service as target
   - Configures health checks
   - Manages security groups
4. Returns ALB hostname in Ingress status
5. Manages lifecycle (creates/deletes ALB as needed)
```

---

## Part 6: Interview Questions & Answers

### Q1: "Explain CRDs vs Deployments. When would you define a CRD?"

**Answer:**
- **Deployment:** Kubernetes built-in, manages stateless workloads
- **CRD:** Custom resource for domain-specific concepts

**When to define CRD:**
- Managing stateful systems (databases, caches, queues)
- Abstracting infrastructure (Database CRD → abstracts RDS, PostgreSQL, MongoDB)
- Multi-resource coordination (single CRD → provisions multiple resources)
- Example: `Database` CRD is better than 10 individual manifests

---

### Q2: "Design a CRD for a Cache resource (like Redis)."

**Answer:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: caches.mycompany.com
spec:
  group: mycompany.com
  names:
    kind: Cache
    plural: caches
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - engine
            - capacity
            properties:
              engine:
                type: string
                enum: ["redis", "memcached"]
              capacity:
                type: string
                pattern: "^[0-9]+(Gi|Mi)$"  # 5Gi, 512Mi
              replicas:
                type: integer
                minimum: 1
                maximum: 3
              persistenceEnabled:
                type: boolean
              ttl:
                type: integer  # seconds
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Creating", "Ready", "Failed"]
              endpoint:
                type: string
              availableReplicas:
                type: integer
    subresources:
      status: {}
```

---

### Q3: "Walk through the operator reconciliation loop for a Database operator."

**Answer:**
```
1. User creates:
   apiVersion: mycompany.com/v1
   kind: Database
   metadata:
     name: prod-db
   spec:
     engine: postgresql
     replicas: 3
     backup: {enabled: true}

2. Operator watches all Database resources

3. Operator triggers reconciliation:
   - Loop detects new Database "prod-db"
   - Gets spec from CRD
   - Checks current state: RDS instance exists? No.

4. Operator acts (remediation):
   - Create RDS instance (postgresql)
   - Create 2 read replicas
   - Enable automated backups
   - Create Secret with connection string

5. Operator updates status:
   status:
     phase: Running
     endpoint: prod-db.c123.us-east-1.rds.amazonaws.com
     readyReplicas: 3
     lastReconcileTime: 2024-01-15T10:30:00Z

6. Reconciliation loop continues:
   - If user updates spec.replicas to 5:
     → Operator scales up to 5 replicas
   - If external RDS instance deleted:
     → Operator detects drift, recreates it
   - Every 30 seconds (default), reconcile runs again (idempotent)
```

---

### Q4: "How would you handle backup/restore with an operator?"

**Answer:**
```yaml
# 1. Define Backup resource
apiVersion: mycompany.com/v1
kind: Backup
metadata:
  name: prod-db-backup-20240115
spec:
  database:
    name: prod-db
  retention: 30d  # Keep for 30 days

---
# 2. Define Restore resource
apiVersion: mycompany.com/v1
kind: Restore
metadata:
  name: restore-prod-db
spec:
  sourceBackup:
    name: prod-db-backup-20240115
  targetDatabase:
    name: prod-db-restored

---
# Operator logic:
# For Backup:
# 1. Create snapshot of database
# 2. Upload to S3
# 3. Update status with S3 location
# 4. Auto-cleanup after retention period

# For Restore:
# 1. Retrieve backup from S3
# 2. Provision new database from backup
# 3. Update connection secrets
# 4. Update status when ready
```

---

### Q5: "Compare operator approach vs Helm charts for deploying complex applications."

**Answer:**

| Aspect | Operator | Helm Chart |
|--------|----------|-----------|
| **Setup** | CRD + Controller | Just templates |
| **Day 2 Ops** | Auto-scaling, upgrades, healing | Manual |
| **State Management** | Full reconciliation | Declarative, no state tracking |
| **Complexity** | Higher (need controller code) | Lower (YAML templating) |
| **Ideal For** | Stateful systems (DBs, queues) | Stateless workloads (web apps) |
| **Debugging** | Check controller logs | Check pod logs, events |

**Use Helm for:**
- Web apps, APIs
- Quick deployments
- Simple configurations

**Use Operator for:**
- Databases (PostgreSQL, MongoDB)
- Message queues (Kafka, RabbitMQ)
- Complex infrastructure (Prometheus, Istio)

---

## Part 7: Hands-On: Build a Simple Operator

### 7.1 Domain: Configuration Watcher Operator

**Problem:** Watch ConfigMap changes, restart pods using that ConfigMap

**Step 1: Define CRD**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: configwatchers.mycompany.com
spec:
  group: mycompany.com
  names:
    kind: ConfigWatcher
    plural: configwatchers
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - configMapName
            - labelSelector
            properties:
              configMapName:
                type: string
              labelSelector:
                type: object
          status:
            type: object
    subresources:
      status: {}
```

**Step 2: Create CRD**
```bash
kubectl apply -f configwatcher-crd.yaml
```

**Step 3: Define Operator (Pseudocode - Python)**
```python
import kopf  # Kubernetes Operator Framework

@kopf.on.event('mycompany.com', 'v1', 'configwatchers')
def watch_config(event, **kwargs):
    configwatcher = event['object']
    namespace = configwatcher['metadata']['namespace']
    configmap_name = configwatcher['spec']['configMapName']
    label_selector = configwatcher['spec']['labelSelector']
    
    # Watch ConfigMap for changes
    configmap = k8s.get_configmap(namespace, configmap_name)
    
    # Find all pods with matching labels
    pods = k8s.list_pods(namespace, label_selector)
    
    # Restart matching pods
    for pod in pods:
        k8s.delete_pod(pod)  # K8s recreates via Deployment
    
    # Update status
    configwatcher['status']['podRestartCount'] = len(pods)
    k8s.update_status(configwatcher)
```

**Step 4: Use CRD**
```yaml
apiVersion: mycompany.com/v1
kind: ConfigWatcher
metadata:
  name: app-config-watcher
spec:
  configMapName: app-config
  labelSelector:
    app: web-server
```

**Result:** Any change to `app-config` ConfigMap → all pods with `app: web-server` restarted automatically

---

## Part 8: Self-Assessment

- [ ] Understand CRD structure (group, kind, scope, schema)
- [ ] Can write OpenAPI validation schemas
- [ ] Know when to use finalizers and webhooks
- [ ] Understand operator pattern (CRD + Controller)
- [ ] Can explain reconciliation loop
- [ ] Know workspace operators (Jaeger, cert-manager, ALB controller)
- [ ] Can design simple CRDs for domain problems
- [ ] Interview confidence: Design CRD for new system

---

## Key Takeaways

1. **CRD = API Extension** - Make Kubernetes understand your domain
2. **Operator = Automation** - Controller implements domain knowledge
3. **Reconciliation Loop** - Core pattern: observe → compare → act → update status
4. **Idempotency** - Key to reliability (safe to rerun)
5. **Workspace Examples** - Study Jaeger, cert-manager, ALB controller for patterns
6. **Day 2 Ops** - Operators shine for stateful systems and complex lifecycle

---

## Study Plan (Next 2 Hours)

1. **Theory (30 min):** Read CRD fundamentals, operator pattern
2. **Workspace Review (30 min):** Study Jaeger Operator, cert-manager
3. **Design Exercise (30 min):** Design CRD for domain problem
4. **Interview Prep (30 min):** Practice Q1-Q5 answers

**Next Module:** Master EKS architecture (control plane, node groups, IRSA)
