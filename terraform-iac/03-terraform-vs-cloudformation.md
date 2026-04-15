# 03 - Terraform vs CloudFormation: When to Use Each

## Side-by-Side Comparison

| Feature | Terraform | CloudFormation |
|---------|-----------|----------------|
| **Cloud Support** | Multi-cloud (AWS, Azure, GCP, 3000+ providers) | AWS only |
| **Syntax** | HCL (clean, readable) | JSON/YAML (verbose) |
| **State Management** | Manual (S3 + DynamoDB) | AWS managed (automatic) |
| **Module System** | Terraform modules (rich ecosystem) | Nested stacks | **Learning Curve** | Medium | Steeper |
| **New AWS Services** | Wait for provider update | Day-0 support (immediate) |
| **Community** | Large open-source community | AWS support |
| **Cost** | Free (open source) | Free |
| **Import Resources** | Yes (terraform import) | Yes (via CloudFormation) |
| **Drift Detection** | Yes (terraform plan) | Yes (native feature) |
| **Testing** | terraform validate, external tools | cfn-lint, CloudFormation validate |

---

## Same Infrastructure: Terraform vs CloudFormation

### Terraform Example (EKS Cluster)

```hcl
# main.tf
provider "aws" {
  region = var.region
}

resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.27"

  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true
    endpoint_public_access  = true
    security_group_ids      = [aws_security_group.cluster.id]
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator"]

  tags = {
    Name        = var.cluster_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.cluster_name}-nodes"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id

  scaling_config {
    desired_size = 6
    max_size     = 12
    min_size     = 3
  }

  instance_types = [var.node_instance_type]

  tags = {
    Name = "${var.cluster_name}-nodes"
  }
}

output "cluster_endpoint" {
  value = aws_eks_cluster.main.endpoint
}
```

**Lines of code:** ~45 lines

### CloudFormation Equivalent (Same EKS Cluster)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'EKS Cluster'

Parameters:
  ClusterName:
    Type: String
    Default: prod-eks
  NodeInstanceType:
    Type: String
    Default: c5.xlarge

Resources:
  EKSCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: !Ref ClusterName
      Version: '1.27'
      RoleArn: !GetAtt EKSClusterRole.Arn
      ResourcesVpcConfig:
        SubnetIds:
          - !Ref PrivateSubnet1
          - !Ref PrivateSubnet2
          - !Ref PrivateSubnet3
        EndpointPrivateAccess: true
        EndpointPublicAccess: true
        SecurityGroupIds:
          - !Ref ClusterSecurityGroup
      Logging:
        ClusterLogging:
          EnabledTypes:
            - Type: api
            - Type: audit
            - Type: authenticator
      Tags:
        - Key: Name
          Value: !Ref ClusterName
        - Key: Environment
          Value: production
        - Key: ManagedBy
          Value: CloudFormation

  NodeGroup:
    Type: AWS::EKS::Nodegroup
    DependsOn: EKSCluster
    Properties:
      ClusterName: !Ref EKSCluster
      NodegroupName: !Sub '${ClusterName}-nodes'
      NodeRole: !GetAtt NodeRole.Arn
      Subnets:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      ScalingConfig:
        DesiredSize: 6
        MaxSize: 12
        MinSize: 3
      InstanceTypes:
        - !Ref NodeInstanceType
      Tags:
        Name: !Sub '${ClusterName}-nodes'

Outputs:
  ClusterEndpoint:
    Value: !GetAtt EKSCluster.Endpoint
    Export:
      Name: !Sub '${AWS::StackName}-ClusterEndpoint'
```

**Lines of code:** ~65 lines

**Observation:** Terraform is more concise; CloudFormation is more verbose but has built-in functions.

---

## When to Use Terraform

### ✅ Use Terraform When:

1. **Multi-cloud requirements**
   - Need to manage AWS + Azure + GCP
   - Want provider flexibility
   - Future cloud migration possible

2. **Module ecosystem**
   - Want to use community modules
   - Need reusable components across projects
   - Terraform Registry has 3000+ modules

3. **Cleaner syntax preference**
   - HCL is more readable than YAML
   - Less verbose
   - Better IDE support

4. **Existing Terraform experience**
   - Team knows Terraform
   - Already have Terraform modules
   - Migration cost too high

5. **Need advanced features**
   - Complex loops and conditionals
   - Dynamic blocks
   - Provider-specific features (Kubernetes, Helm)

### Real Example:

```hcl
# Terraform: Create 3 subnets with dynamic block
resource "aws_subnet" "private" {
  count      = 3
  vpc_id     = aws_vpc.main.id
  cidr_block = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}
```

**Much cleaner than CloudFormation equivalent!**

---

## When to Use CloudFormation

### ✅ Use CloudFormation When:

1. **AWS-only infrastructure**
   - No plans for multi-cloud
   - All resources are AWS services
   - Want AWS-native tooling

2. **Day-0 service support**
   - New AWS services supported immediately
   - No waiting for Terraform provider updates
   - Critical for early adopters

3. **AWS-managed state**
   - Don't want to manage S3 backend
   - Simplified operations
   - AWS handles versioning and locking

4. **AWS ecosystem integration**
   - Using AWS Service Catalog
   - Using AWS Control Tower
   - Using AWS Organizations
   - Need StackSets for multi-account

5. **Compliance requirements**
   - Some enterprises require AWS-native tools
   - Auditing and compliance easier
   - CloudFormation Guard for policy-as-code

### Real Example:

```yaml
# CloudFormation: New AWS service (supports day-0)
Resources:
  NewAWSService:
    Type: AWS::NewService::Resource  # Available immediately
    Properties:
      # Supported on release day
```

With Terraform, you'd wait for provider update (days/weeks).

---

## Using Both Together

### Hybrid Approach

```
Terraform manages:
├── VPC and networking (better module support)
├── EKS cluster (community modules available)
├── IAM roles (shared across resources)
└── S3, DynamoDB (state backend itself)

CloudFormation manages:
├── Service Catalog portfolios
├── AWS Control Tower guardrails
├── Organization-wide security baselines (StackSets)
└── New AWS services not yet in Terraform
```

### Accessing CloudFormation Outputs in Terraform

```hcl
# Terraform data source reads CloudFormation stack output
data "aws_cloudformation_stack" "network" {
  name = "network-stack"
}

resource "aws_eks_cluster" "main" {
  vpc_config {
    # Use CloudFormation output
    subnet_ids = split(",", data.aws_cloudformation_stack.network.outputs["SubnetIds"])
  }
}
```

---

## State Management Comparison

### Terraform State

```
Local State:
├── terraform.tfstate (JSON file in directory)
├── No collaboration (single developer only)
├── Risk of data loss
└── No locking

Remote State:
├── S3 bucket: company-terraform-state
├── DynamoDB table: terraform-locks
├── Versioning enabled (rollback capability)
├── Encryption at rest
├── Team collaboration (shared state)
└── State locking (prevents conflicts)

State contains:
- Resource IDs
- Resource attributes
- Dependencies
- Metadata
```

### CloudFormation State

```
AWS Managed State:
├── Stored in AWS (you don't see it)
├── Automatic versioning
├── Automatic locking
├── No S3 bucket to manage
├── No DynamoDB table needed
└── Free

Stack contains:
- Resource logical IDs
- Resource physical IDs
- Parameters
- Outputs
- Template version
```

**Key difference:** With Terraform, you manage state. With CloudFormation, AWS manages it for you.

---

## Decision Matrix: Terraform vs CloudFormation

### Scenario: Deploy EKS across 4 environments

**Choose Terraform if:**
- ✅ Want reusable modules for VPC, EKS, security groups
- ✅ Team experienced with Terraform
- ✅ May need to deploy to other clouds (Azure, GCP)
- ✅ Want community modules (terraform-aws-modules/eks)
- ✅ Prefer HCL syntax

**Choose CloudFormation if:**
- ✅ AWS-only (no multi-cloud plans)
- ✅ Want AWS-managed state (simpler ops)
- ✅ Need StackSets for multi-account deployment
- ✅ Using AWS Control Tower or Service Catalog
- ✅ Team experienced with CloudFormation

**My recommendation:** Start with Terraform for flexibility. Use CloudFormation only when Terraform doesn't support a feature or you need StackSets.

---

## Common Interview Questions

### Q: "Have you used both Terraform and CloudFormation? When do you choose each?"

**Answer:**
"Yes, I've used both extensively. I primarily use Terraform for core infrastructure because of its multi-cloud support, cleaner HCL syntax, and excellent module ecosystem. Terraform's community has built reusable modules for common patterns (VPCs, EKS clusters), which save time. However, I use CloudFormation for specific scenarios: when I need immediate support for new AWS services, when deploying organization-wide baselines via StackSets, or when integrating with AWS Service Catalog. In some projects, I use both: Terraform for VPC and EKS, CloudFormation for organization governance. They interoperate well - Terraform can read CloudFormation outputs via data sources."

### Q: "What are the trade-offs between Terraform and CloudFormation?"

**Answer:**
"**Terraform advantages:**
- Multi-cloud portability (can move to Azure/GCP)
- Better syntax (HCL vs YAML/JSON)
- Rich module ecosystem (terraform-aws-modules)
- Better community support

**Terraform disadvantages:**
- Must manage state (S3 + DynamoDB setup)
- New AWS services lag behind (wait for provider)
- State corruption risk (if not careful)

**CloudFormation advantages:**
- AWS-managed state (simpler operations)
- Day-0 support for new AWS services
- Deep AWS integration (Control Tower, Service Catalog)
- StackSets for multi-account deployments

**CloudFormation disadvantages:**
- AWS-only (vendor lock-in)
- More verbose syntax
- Less active community
- Nested stacks more complex than Terraform modules

**My approach:** Use Terraform as default, CloudFormation when needed for AWS-specific features."

