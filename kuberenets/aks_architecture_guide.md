# Azure AKS - Architecture & Comparison with EKS

## Module Overview

**Learning Objectives:**
- Understand AKS control plane & architecture
- Master node pools (system vs user pools)
- Learn managed identity vs IRSA
- Azure network integration (CNI, NSGs, UDRs)
- Compare AKS vs EKS design patterns
- Interview scenarios for multi-cloud environments

**Estimated Study Time:** 8-10 hours
**Context:** Workspace focuses on AWS EKS, but multi-cloud interviewing requires AKS knowledge

---

## Part 1: AKS Control Plane

### 1.1 AKS Architecture Overview

```
┌─────────────────────────────────────────────┐
│      Azure Kubernetes Service (AKS)         │
├─────────────────────────────────────────────┤
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │  Control Plane (Managed by Azure)   │   │
│  │  - kube-apiserver (HA, multi-region)│   │
│  │  - etcd (managed, encrypted)        │   │
│  │  - Controllers, Scheduler           │   │
│  │  - SLA: 99.5% (standard) to 99.95%  │   │
│  │    (with uptime SLA)                │   │
│  └─────────────────────────────────────┘   │
│                 ↓                           │
│  ┌─────────────────────────────────────┐   │
│  │  Node Pools (VMs in customer Azure  │   │
│  │  subscription)                      │   │
│  │  - System node pool (1+)            │   │
│  │  - User node pools (0+)             │   │
│  └─────────────────────────────────────┘   │
│                                             │
└─────────────────────────────────────────────┘
```

**Key Difference from EKS:**
- Control plane in **Azure-managed subscription** (not customer subscription)
- Node pools run in **customer subscription** (you see/manage VMs)
- Costs: Control plane + node compute (pay per VM hour)

### 1.2 AKS vs EKS Control Plane Comparison

| Aspect | AKS | EKS |
|--------|-----|-----|
| **Location** | Managed by Azure (separate subscription) | Managed by AWS (separate account) |
| **Multi-region HA** | Multi-region (automatic failover possible) | Multi-AZ within region |
| **Encryption at Rest** | By default (via platform-managed keys) | Optional (requires KMS setup) |
| **Audit Logging** | Azure Monitor, Azure Audit Logs | CloudWatch Logs |
| **RBAC Integration** | Native Azure RBAC (not K8s RBAC) | IAM (not K8s RBAC) |
| **SLA** | 99.5% (free tier), 99.95% (paid) | 99.95% (always) |
| **API Access** | Public/private endpoints | Public/private endpoints |
| **Price** | Control plane = ~$75/month + compute | Control plane = ~$73/month + compute |

### 1.3 AKS Cluster Creation (Terraform)

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  name                = "my-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  kubernetes_version  = "1.28.0"

  # Default node pool (system pool)
  default_node_pool {
    name            = "system"
    node_count      = 3
    vm_size         = "Standard_D2s_v3"  # Azure VM size
    availability_zones = ["1", "2", "3"]  # Zones for HA
    
    # Node pool labels
    node_labels = {
      workload = "system"
    }
    
    # Taints
    node_taints = ["system:NoSchedule"]  # Only system pods here
  }

  # Network plugin (Azure CNI recommended)
  network_profile {
    network_plugin = "azure"  # or "kubenet" (simpler but limited)
    network_policy = "azure"  # Enable Network Policy
    
    # Pod CIDR (for overlay networks)
    pod_cidr    = "10.244.0.0/16"
    service_cidr = "10.0.0.0/16"
    dns_service_ip = "10.0.0.10"
  }

  # Managed identity (instead of service principal)
  identity {
    type = "SystemAssigned"
  }

  api_server_authorized_ip_ranges = ["0.0.0.0/0"]  # API access

  # Enable monitoring
  monitor_metrics {
    enabled = true
  }

  # Role-based access (Azure AD)
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
    admin_group_object_ids = [azurerm_client_config.current.object_id]
  }

  tags = {
    Environment = "prod"
  }
}

# User node pool (for application workloads)
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  node_count            = 3
  vm_size               = "Standard_D4s_v3"
  availability_zones    = ["1", "2", "3"]

  node_labels = {
    workload = "application"
  }

  auto_scaling_enabled  = true
  min_count             = 1
  max_count             = 10
}
```

---

## Part 2: Node Pools

### 2.1 System Node Pool

**Purpose:** Runs Azure system add-ons
- coredns (DNS)
- metrics-server
- kube-proxy
- Container networking agent

**Requirements:**
- At least 1 system pool per cluster
- Minimum 2 nodes (for HA)
- Usually tainted with `system:NoSchedule`
- Recommended: 3 nodes across 3 zones

**Example:**
```hcl
default_node_pool {
  name            = "system"
  node_count      = 3
  vm_size         = "Standard_D2s_v3"  # Small VMs sufficient
  availability_zones = ["1", "2", "3"]
  
  node_taints = [
    {
      key    = "system"
      value  = "reserved"
      effect = "NoSchedule"
    }
  ]
}
```

### 2.2 User Node Pools

**Purpose:** Runs application workloads

**Multiple pools for different workload types:**
```hcl
# GPU node pool (for ML workloads)
resource "azurerm_kubernetes_cluster_node_pool" "gpu" {
  name                  = "gpu"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_NC6s_v3"  # GPU VM
  node_count            = 2

  node_labels = {
    workload      = "ml"
    gpu           = "true"
    gpu_type      = "nvidia-tesla-v100"
  }

  node_taints = [
    {
      key    = "gpu"
      value  = "true"
      effect = "NoSchedule"
    }
  ]

  auto_scaling_enabled = true
  min_count            = 0
  max_count            = 5
}

# High-memory node pool (for data processing)
resource "azurerm_kubernetes_cluster_node_pool" "memory" {
  name                  = "memory"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_E16s_v3"  # 128 GB RAM
  node_count            = 2

  node_labels = {
    workload = "high-memory"
  }

  auto_scaling_enabled = true
  min_count            = 1
  max_count            = 10
}
```

**Pod scheduling to specific pool:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  nodeSelector:
    gpu: "true"
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: app
    image: ml-app:latest
```

### 2.3 Node Pool Scaling

**Manual Scaling:**
```bash
az aks nodepool scale \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name user \
  --node-count 5
```

**Auto Scaling:**
```hcl
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  auto_scaling_enabled = true
  min_count            = 1
  max_count            = 20
}
```

**Cluster Autoscaler:** Works same as EKS
- Watches for pending pods
- Scales up node pool
- Scales down idle nodes

---

## Part 3: Managed Identity vs Service Principal

### 3.1 Service Principal (Legacy)

**Not recommended anymore, but good to know:**

```bash
# Create service principal
az ad sp create-for-rbac --name my-aks-sp

# Output:
# {
#   "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#   "password": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
#   "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# }

# Cluster uses service principal credentials
# Credentials stored in kubeconfig, accessible to humans
# Risk: Credentials can be leaked, hard to rotate
```

### 3.2 Managed Identity (Modern)

**Azure-managed identity, similar to AWS IRSA:**

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  # ... cluster config
  
  identity {
    type = "SystemAssigned"  # Azure creates & manages identity
  }
}
```

**Cluster gets system-assigned managed identity:**
- Azure automatically creates identity
- No credentials to manage
- No kubeconfig secret needed
- Automatic token refresh

**Pod-level Managed Identity (Workload Identity):**

```hcl
# Create user-assigned managed identity
resource "azurerm_user_assigned_identity" "app" {
  name                = "app-identity"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
}

# Assign role (e.g., read from Key Vault)
resource "azurerm_role_assignment" "app_keyvault" {
  scope              = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id       = azurerm_user_assigned_identity.app.principal_id
}

# Federate identity with K8s ServiceAccount
resource "azurerm_federated_identity_credential" "app" {
  name                = "app-fed-cred"
  resource_group_name = azurerm_resource_group.main.name
  parent_id           = azurerm_user_assigned_identity.app.id
  audience            = ["api://AzureADTokenExchange"]
  
  # This K8s ServiceAccount
  issuer   = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject  = "system:serviceaccount:default:app"
}
```

**Pod Setup:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: default
  annotations:
    azure.workload.identity/client-id: "xxx-xxx-xxx"

---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app
  containers:
  - name: app
    image: app:latest
    # SDK automatically uses managed identity
```

---

## Part 4: Azure Network Integration

### 4.1 Azure CNI vs Kubenet

| Aspect | Azure CNI | Kubenet |
|--------|-----------|---------|
| **Pod IP Source** | Azure VNet (same as nodes) | Internal (bridge network) |
| **Networking** | Native Azure networking | Overlay |
| **Performance** | Higher (direct VNet integration) | Lower (overlay overhead) |
| **IP Capacity** | Limited (VNet CIDR) | Higher (separate pod CIDR) |
| **Features** | Network policies, private IPs | Basic routing |
| **Recommended** | Yes (enterprise) | No (legacy) |

### 4.2 Azure CNI Configuration

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  network_profile {
    network_plugin    = "azure"
    network_policy    = "azure"  # Enable Network Policy
    service_cidr      = "10.0.0.0/16"
    dns_service_ip    = "10.0.0.10"
    pod_cidr          = "10.244.0.0/16"  # For overlay (optional)
    
    # Load balancer settings
    load_balancer_sku = "standard"  # Standard Load Balancer
  }
}
```

### 4.3 Network Policies (Azure NSG-based)

**Similar to Kubernetes NetworkPolicy but uses Azure NSGs:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Result:** Azure creates NSG rules enforcing the policy

### 4.4 User-Defined Routes (UDRs)

**Custom routing for advanced networking:**

```hcl
resource "azurerm_route_table" "custom" {
  name                = "custom-routes"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  route {
    name           = "to-on-prem"
    address_prefix = "192.168.0.0/16"
    next_hop_type  = "VirtualAppliance"
    next_hop_in_ip_address = "10.0.1.1"  # NVA
  }
}

# Associate UDR with subnet
resource "azurerm_subnet_route_table_association" "main" {
  subnet_id      = azurerm_subnet.worker.id
  route_table_id = azurerm_route_table.custom.id
}
```

---

## Part 5: Azure AD Integration

### 5.1 RBAC via Azure AD

**Kubernetes RBAC is NOT used; Azure AD RBAC instead:**

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
    admin_group_object_ids = [azurerm_client_config.current.object_id]
  }
}
```

**Authorization flow:**
1. User logs in with Azure AD
2. `kubeconfig` uses Azure AD token
3. Kubernetes API Server validates token with Azure AD
4. Azure AD RBAC determines permissions (not K8s RBAC)

**Example: Grant user cluster-admin via Azure AD:**
```hcl
resource "azurerm_role_assignment" "dev_cluster_admin" {
  scope              = azurerm_kubernetes_cluster.main.id
  role_definition_name = "Azure Kubernetes Service Cluster Admin Role"
  principal_id       = data.azuread_user.dev.object_id
}
```

### 5.2 Workload Identity (Pod Identity)

**Grant pods permissions via Azure AD:**

```hcl
# Create workload identity
resource "azurerm_user_assigned_identity" "workload" {
  name                = "app-workload"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
}

# Grant permission (e.g., write to blob storage)
resource "azurerm_role_assignment" "workload_storage" {
  scope              = azurerm_storage_account.main.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id       = azurerm_user_assigned_identity.workload.principal_id
}

# Federate with K8s ServiceAccount
resource "azurerm_federated_identity_credential" "workload" {
  name                = "workload-fed"
  resource_group_name = azurerm_resource_group.main.name
  parent_id           = azurerm_user_assigned_identity.workload.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject             = "system:serviceaccount:default:app"
}
```

**Pod Uses Workload Identity:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app
  containers:
  - name: app
    image: app:latest
    env:
    - name: AZURE_CLIENT_ID
      value: "xxx-xxx-xxx"  # Use the same client ID from ServiceAccount annotation
```

**Note:** The downward API's `fieldRef` does NOT support bracket notation for accessing specific annotations. Instead, pass the client ID as a literal value or use the Azure Workload Identity webhook to inject it automatically. The webhook will add the `AZURE_CLIENT_ID` env var automatically when this pod is scheduled.

---

## Part 6: AKS Add-ons & Extensions

### 6.1 Built-in Add-ons

**Installed by default:**
- DNS (CoreDNS)
- Metrics Server
- Azure policy add-on (optional)

**Enable monitoring:**
```hcl
resource "azurerm_kubernetes_cluster" "main" {
  monitor_metrics {
    enabled = true
  }
}
```

### 6.2 Virtual Nodes (Serverless Containers)

**Run pods on ACI (Azure Container Instances) instead of VMs:**

```hcl
resource "azurerm_kubernetes_cluster" "main" {
  aci_connector_linux {
    enabled     = true
    subnet_name = azurerm_subnet.virtual_nodes.name
  }
}
```

**Schedule pod on virtual node:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: virtual-node-pod
spec:
  nodeSelector:
    kubernetes.io/virtual-kubelet: "true"
  tolerations:
  - key: virtual-kubelet.io/provider
    operator: Equal
    value: azure
  containers:
  - name: app
    image: app:latest
```

**Benefits:**
- Serverless: Pay only for what you use
- No node management
- Great for batch, variable workloads

---

## Part 7: AKS vs EKS - Complete Comparison

| Feature | AKS | EKS |
|---------|-----|-----|
| **Cloud** | Azure | AWS |
| **Control Plane Management** | Managed (azure-managed subscription) | Managed (aws-managed account) |
| **Multi-AZ HA** | Across 3 zones (optional, requires uptime SLA) | Default across 3 zones |
| **Identity System** | Managed Identity + Azure AD | IRSA + IAM |
| **Network Plugin** | Azure CNI or Kubenet | Amazon VPC CNI only |
| **Storage** | Azure Disk, Azure Files | EBS, EFS |
| **Monitoring** | Azure Monitor built-in | CloudWatch (needs setup) |
| **Autoscaling** | Cluster Autoscaler | Cluster Autoscaler or Karpenter |
| **Cost (Control Plane)** | Free or $74 (SLA) | ~$73/month |
| **Cost (Compute)** | Pay per VM hour | Pay per EC2 hour |
| **Node Auto-Repair** | Yes (automatic) | No (manual) |
| **Kubernetes RBAC** | No (uses Azure AD RBAC) | Yes (IAM for authentication only) |
| **Network Policies** | Yes (Azure-based) | Yes (CNI-based, e.g., Calico) |

---

## Part 8: Interview Scenarios

### Scenario 1: Design Multi-Cloud Application

**Requirements:**
- Primary: Azure (enterprise partnership)
- Secondary: AWS (disaster recovery)
- Shared codebase, different credentials

**Architecture:**
```yaml
Primary (Azure AKS):
- 3-node system pool
- 3-node app pool (auto-scaling)
- Managed identity for pod access (Azure Storage, Key Vault)
- Azure Monitor + App Insights
- Azure Container Registry (ACR)

Secondary (AWS EKS):
- 3-node default pool (auto-scaling)
- IRSA for pod access (S3, secrets manager)
- CloudWatch for monitoring
- ECR with image replication from ACR

Shared:
- Same Helm charts (cloud-agnostic)
- Terraform modules for both clouds
- Shared container image registry
- Backup/sync between regions
```

### Scenario 2: Troubleshoot Pod Can't Access Key Vault (AKS)

**Problem:** Pod needs access to Azure Key Vault secret

**Diagnosis:**
1. Check workload identity setup:
   ```bash
   kubectl describe sa <pod-sa> -n <namespace>
   # Should have azure.workload.identity/client-id annotation
   ```

2. Check federated credential:
   ```bash
   az identity federated-credential list \
     --resource-group <rg> \
     --identity-name <identity>
   ```

3. Check role assignment:
   ```bash
   az role assignment list \
     --assignee <principal-id> \
     --scope <key-vault-id>
   ```

4. Check pod token:
   ```bash
   kubectl exec <pod> -- printenv | grep AZURE
   # Should have AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_FEDERATED_TOKEN_FILE
   ```

5. Test Key Vault access:
   ```bash
   kubectl exec <pod> -- az keyvault secret show \
     --vault-name my-kv \
     --name my-secret
   ```

---

## Part 9: Migration Scenarios

### EKS to AKS (or vice versa)

**Strategy:**
1. **Replicate infrastructure** (Terraform multi-cloud)
2. **Same Helm charts** (cloud-agnostic)
3. **Database migration** (managed service: RDS → Azure Database)
4. **Image replication** (ECR → ACR)
5. **DNS switchover** (update ingress or external DNS)

**Example Migration:**
```
Week 1: Set up AKS cluster, replicate workloads
Week 2: Test in parallel (EKS + AKS both running)
Week 3: Route 10% traffic to AKS, monitor
Week 4: Route 100% traffic to AKS, retire EKS
```

---

## Part 10: Key Takeaways

1. **AKS Control Plane:** Managed by Azure (separate tenant), no visibility
2. **Node Pools:** System (required) + User (optional, multiple types)
3. **Managed Identity:** Azure-managed identity (like AWS IRSA)
4. **Network:** Azure CNI recommended (vs Kubenet)
5. **RBAC:** Azure AD RBAC (not K8s RBAC)
6. **Monitoring:** Azure Monitor built-in
7. **Multi-cloud:** Use Terraform, Helm charts to share configs

---

## Study Plan (Next 2-3 Hours)

1. **Theory (45 min):** AKS architecture, node pools, managed identity
2. **Comparison (30 min):** AKS vs EKS differences
3. **Configuration (45 min):** Workload identity setup
4. **Multi-cloud (30 min):** Design multi-cloud application
5. **Interview Prep (15 min):** Practice Scenario 1 & 2

**Next Module:** Bare Metal Kubernetes (kubeadm, CNI choice, HA control plane)
