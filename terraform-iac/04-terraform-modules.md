# 04 - Terraform Modules: Reusable Infrastructure Components

## What are Terraform Modules?

Modules are containers for multiple resources that are used together. A module is a collection of `.tf` files in a directory. Every Terraform configuration has at least one module (the root module).

### Why Modules?

```
Without modules:
├── Copy-paste VPC code for dev, qa, prod
├── Duplicate security group rules
├── Hard to maintain (change in 4 places)
└── Error-prone

With modules:
├── Define VPC once in module
├── Reuse module with different parameters
├── Change in one place
└── Consistent across environments
```

---

## Module Structure

```
terraform-aws-modules/
├── vpc/
│   ├── main.tf           # Resources
│   ├── variables.tf      # Input variables
│   ├── outputs.tf        # Output values
│   ├── README.md         # Documentation
│   ├── examples/
│   │   └── complete/     # Usage examples
│   └── versions.tf       # Provider requirements
├── eks/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
└── security-group/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

---

## Creating a VPC Module

### Module Definition (modules/vpc/main.tf)

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}

resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-public-${count.index + 1}"
      Type = "public"
    }
  )
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-private-${count.index + 1}"
      Type = "private"
    }
  )
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-igw"
    }
  )
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-nat-${count.index + 1}"
    }
  )
}

resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-eip-${count.index + 1}"
    }
  )
}
```

### Module Variables (modules/vpc/variables.tf)

```hcl
variable "name" {
  description = "Name prefix for VPC resources"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "tags" {
  description = "Additional tags for all resources"
  type        = map(string)
  default     = {}
}
```

### Module Outputs (modules/vpc/outputs.tf)

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ids" {
  description = "List of NAT Gateway IDs"
  value       = aws_nat_gateway.main[*].id
}
```

---

## Using Modules

### Calling a Module

```hcl
# Root module (main.tf)
module "vpc_dev" {
  source = "./modules/vpc"

  name               = "dev-vpc"
  vpc_cidr           = "10.1.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b"]

  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}

module "vpc_prod" {
  source = "./modules/vpc"

  name               = "prod-vpc"
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
    CriticalData = "true"
  }
}

# Use module outputs
resource "aws_eks_cluster" "dev" {
  name = "dev-eks"

  vpc_config {
    subnet_ids = module.vpc_dev.private_subnet_ids
  }
}

resource "aws_eks_cluster" "prod" {
  name = "prod-eks"

  vpc_config {
    subnet_ids = module.vpc_prod.private_subnet_ids
  }
}
```

---

## Module Sources

### Local Path

```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

### Git Repository

```hcl
module "vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0"
}
```

### Terraform Registry

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Terraform   = "true"
    Environment = "prod"
  }
}
```

---

## Module Versioning

### Pin Module Versions

```hcl
# Bad: No version (uses latest)
module "vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc"
}

# Good: Pinned version
module "vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0"
}

# Best: Terraform Registry with version constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # Allow 5.x updates, not 6.x
}
```

**Version Constraints:**
```hcl
version = "1.2.0"      # Exact version
version = ">= 1.2.0"   # Greater than or equal
version = "~> 1.2"     # Allow 1.x, not 2.x
version = ">= 1.2.0, < 2.0.0"  # Range
```

---

## Module Composition (Modules Calling Modules)

```hcl
# modules/eks-complete/main.tf
module "vpc" {
  source = "../vpc"

  name               = var.cluster_name
  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones
}

module "eks" {
  source = "../eks"

  cluster_name = var.cluster_name
  subnet_ids   = module.vpc.private_subnet_ids
  vpc_id       = module.vpc.vpc_id
}

module "node_group" {
  source = "../eks-node-group"

  cluster_name   = module.eks.cluster_name
  subnet_ids     = module.vpc.private_subnet_ids
  instance_types = var.node_instance_types
}
```

**Benefit:** Compose complex infrastructure from simple modules.

---

## Common Interview Questions

### Q: "How do you design Terraform modules?"

**Answer:**
"I design modules to be reusable, configurable, and composable. Each module has three files: main.tf (resources), variables.tf (inputs), and outputs.tf (outputs). I follow these principles:

1. **Single responsibility:** Each module does one thing (VPC module creates VPC, not EKS).
2. **Configurable:** Expose important parameters as variables with sensible defaults.
3. **Outputs:** Return useful values (IDs, ARNs) that other modules need.
4. **No hard-coded values:** Everything configurable via variables.
5. **Versioned:** Pin module versions for stability.

Example: My VPC module accepts CIDR, AZs, and tags as inputs. It creates VPC, subnets, NAT gateways, and returns subnet IDs as outputs. Other modules (EKS, RDS) consume these outputs."

### Q: "How do you version and test modules?"

**Answer:**
"Modules are stored in a separate Git repository with semantic versioning (v1.0.0, v1.1.0, v2.0.0). Each version is a Git tag. When using modules, I pin to specific versions to prevent breaking changes:

```hcl
module \"vpc\" {
  source = \"git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0\"
}
```

For testing, I use terraform validate and terraform plan in a test environment. I also have example configurations in the `examples/` directory that I test before releasing new versions. Major version bumps (v2.0.0) indicate breaking changes; minor bumps (v1.1.0) are backwards-compatible."

### Q: "What's the difference between a module and a resource?"

**Answer:**
"A resource is a single infrastructure component (one EC2 instance, one S3 bucket). A module is a collection of resources that work together. For example, a VPC module contains VPC resource, subnet resources, internet gateway, NAT gateways, route tables - all the components needed for a functional VPC. Modules provide reusability: define VPC once, use it many times with different parameters. Resources are building blocks; modules are blueprints."

