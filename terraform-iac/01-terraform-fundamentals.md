# 01 - Terraform Fundamentals: Infrastructure as Code

## What is Terraform?

Terraform is an open-source Infrastructure as Code (IaC) tool by HashiCorp that allows you to define and provision infrastructure using declarative configuration files. It's **cloud-agnostic**, meaning you can manage AWS, Azure, GCP, and many other providers with the same tool.

### Key Concepts

```
Terraform Workflow:
1. Write      → Define infrastructure in .tf files (HCL)
2. Plan       → Preview changes (terraform plan)
3. Apply      → Create/update infrastructure (terraform apply)
4. Manage     → Track state, make changes iteratively
5. Destroy    → Remove infrastructure (terraform destroy)
```

---

## Terraform Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Your Terraform Configuration (.tf files)              │
│  ├── main.tf                                           │
│  ├── variables.tf                                      │
│  ├── outputs.tf                                        │
│  └── terraform.tfvars                                  │
└─────────────────────────────────────────────────────────┘
          ↓ (terraform plan)
┌─────────────────────────────────────────────────────────┐
│  Terraform Core (Engine)                               │
│  - Parses HCL configuration                            │
│  - Builds dependency graph                             │
│  - Compares desired state vs current state             │
└─────────────────────────────────────────────────────────┘
          ↓ (calls provider APIs)
┌─────────────────────────────────────────────────────────┐
│  Providers (AWS, Azure, GCP, etc.)                     │
│  - AWS Provider → Calls AWS APIs                       │
│  - Azure Provider → Calls Azure APIs                   │
│  - Kubernetes Provider → Calls K8s APIs                │
└─────────────────────────────────────────────────────────┘
          ↓ (creates/updates)
┌─────────────────────────────────────────────────────────┐
│  Real Infrastructure                                    │
│  - EC2 instances                                        │
│  - VPCs, subnets, security groups                      │
│  - EKS clusters                                         │
│  - S3 buckets, DynamoDB tables                         │
└─────────────────────────────────────────────────────────┘
          ↓ (tracks in)
┌─────────────────────────────────────────────────────────┐
│  Terraform State (terraform.tfstate)                   │
│  - JSON file mapping config to real resources          │
│  - Stored locally or remotely (S3)                     │
│  - Contains resource IDs, attributes                   │
└─────────────────────────────────────────────────────────┘
```

---

## HCL Syntax (HashiCorp Configuration Language)

### Basic Resource Definition

```hcl
# Provider configuration
provider "aws" {
  region = "us-east-1"
  profile = "default"
}

# Resource definition
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name        = "WebServer"
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Data source (query existing resources)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Variable definition
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

# Output definition
output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web_server.public_ip
}
```

---

## Terraform State Management

### Local State (Default)

```hcl
# terraform.tfstate stored in current directory
# Works for:
# - Single developer
# - Learning/testing
# - Throwaway environments

# Problems:
# - Not suitable for teams (no collaboration)
# - No locking (concurrent modifications)
# - Risk of data loss
# - No versioning
```

### Remote State (Production)

```hcl
# Backend configuration (backend.tf)
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/eks/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    
    # Enables state locking
    # Prevents concurrent terraform apply
  }
}

# What happens:
# 1. terraform init downloads state from S3
# 2. terraform plan/apply acquires lock in DynamoDB
# 3. If another user tries to apply, they get:
#    "Error: Error acquiring the state lock"
# 4. After apply completes, lock is released
# 5. State is uploaded back to S3
```

**Benefits of Remote State:**
- ✅ Team collaboration (shared state)
- ✅ State locking (prevents conflicts)
- ✅ Versioning (S3 versioning enabled)
- ✅ Encryption at rest
- ✅ Backup and disaster recovery

---

## Variables and Outputs

### Variable Types

```hcl
# String variable
variable "region" {
  type    = string
  default = "us-east-1"
}

# Number variable
variable "instance_count" {
  type    = number
  default = 3
}

# Boolean variable
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List variable
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Map variable
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t3.micro"
    prod = "c5.xlarge"
  }
}

# Object variable (complex)
variable "eks_config" {
  type = object({
    cluster_name    = string
    cluster_version = string
    node_count      = number
    node_type       = string
  })
  default = {
    cluster_name    = "prod-eks"
    cluster_version = "1.27"
    node_count      = 3
    node_type       = "t3.large"
  }
}
```

### Using Variables

```hcl
# Reference variable
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  count         = var.instance_count
  
  availability_zone = var.availability_zones[0]
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}

# Variable precedence (highest to lowest):
# 1. Command line: -var="instance_type=t3.large"
# 2. terraform.tfvars file
# 3. Environment variables: TF_VAR_instance_type
# 4. Default value in variable definition
```

### Outputs

```hcl
# Simple output
output "instance_id" {
  value = aws_instance.web.id
}

# List output
output "instance_ips" {
  value = aws_instance.web[*].public_ip
}

# Complex output
output "eks_cluster_info" {
  value = {
    cluster_id       = aws_eks_cluster.main.id
    cluster_endpoint = aws_eks_cluster.main.endpoint
    cluster_version  = aws_eks_cluster.main.version
  }
}

# Sensitive output (not displayed in terminal)
output "database_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

---

## Data Sources

Data sources query existing infrastructure without managing it:

```hcl
# Query latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Query existing VPC
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["MainVPC"]
  }
}

# Query availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Use data source in resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  subnet_id     = data.aws_subnet.public[0].id
  vpc_security_group_ids = [data.aws_security_group.web.id]
}
```

---

## Lifecycle Management

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  
  lifecycle {
    # Create new resource before destroying old
    create_before_destroy = true
    
    # Prevent accidental deletion
    prevent_destroy = true
    
    # Ignore changes to specific attributes
    ignore_changes = [
      tags["LastModified"],
      user_data
    ]
  }
}
```

**Use cases:**
- `create_before_destroy` - Zero-downtime replacements
- `prevent_destroy` - Protect critical resources (databases)
- `ignore_changes` - Ignore external modifications

---

## Terraform Commands

### Essential Commands

```bash
# Initialize working directory (downloads providers)
terraform init

# Validate configuration syntax
terraform validate

# Format code (standard style)
terraform fmt

# Preview changes (dry-run)
terraform plan
terraform plan -out=tfplan  # Save plan to file

# Apply changes
terraform apply
terraform apply tfplan      # Apply saved plan
terraform apply -auto-approve  # Skip confirmation

# Destroy infrastructure
terraform destroy
terraform destroy -target=aws_instance.web  # Destroy specific resource

# Show current state
terraform show

# List resources in state
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0

# Remove resource from state (doesn't delete actual resource)
terraform state rm aws_instance.web

# Refresh state (sync with real infrastructure)
terraform refresh
```

### Advanced Commands

```bash
# Taint resource (mark for recreation)
terraform taint aws_instance.web

# Untaint resource
terraform untaint aws_instance.web

# Migrate state
terraform state mv aws_instance.web aws_instance.web_new

# Pull state to stdout
terraform state pull

# Push state from file
terraform state push terraform.tfstate

# Unlock state (if locked due to crash)
terraform force-unlock <lock-id>

# Graph dependencies (requires graphviz)
terraform graph | dot -Tpng > graph.png

# Output all outputs
terraform output

# Output specific output
terraform output instance_ip
```

---

## Common Interview Questions

### Q: "What is Terraform state and why is it important?"

**Answer:**
"Terraform state is a JSON file that maps your configuration to real infrastructure resources. It's critical because Terraform uses state to know what resources it manages and their current attributes. Without state, Terraform can't track resources, detect drift, or plan changes. For teams, we use remote state in S3 with DynamoDB locking to enable collaboration and prevent concurrent modifications that could corrupt state."

### Q: "How do you manage secrets in Terraform?"

**Answer:**
"I never store secrets directly in Terraform files or state. Instead, I use AWS Secrets Manager or Parameter Store for secret storage, and reference them via data sources. For example:

```hcl
data \"aws_secretsmanager_secret_version\" \"db_password\" {
  secret_id = \"prod/db/password\"
}

resource \"aws_db_instance\" \"main\" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

Sensitive outputs are marked with `sensitive = true` to prevent display in logs. State files are encrypted at rest in S3 and access is controlled via IAM."

### Q: "What's the difference between `count` and `for_each`?"

**Answer:**
"`count` creates multiple identical resources with numeric indexes (0, 1, 2). `for_each` creates resources based on a map or set with meaningful keys. I prefer `for_each` because it's more resilient to changes. With `count`, if you remove an item from the middle of a list, Terraform recreates all subsequent resources. With `for_each`, only the removed item is destroyed."

```hcl
# count (problematic if list changes)
resource \"aws_instance\" \"web\" {
  count = length(var.subnets)
  subnet_id = var.subnets[count.index]
}

# for_each (better)
resource \"aws_instance\" \"web\" {
  for_each = toset(var.subnets)
  subnet_id = each.value
}
```

### Q: "How do you handle state locking?"

**Answer:**
"State locking prevents concurrent terraform operations that could corrupt state. We use S3 backend with DynamoDB for locking. When terraform apply runs, it acquires a lock in DynamoDB. If another user tries to apply, they get an error. The lock is automatically released when the operation completes. If a crash leaves the state locked, we use `terraform force-unlock` with the lock ID shown in the error message."

