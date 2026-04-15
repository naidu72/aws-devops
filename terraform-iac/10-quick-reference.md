# 10 - Terraform & CloudFormation Quick Reference

## Terraform Commands Cheat Sheet

```bash
# Initialization
terraform init                    # Initialize working directory
terraform init -upgrade           # Upgrade providers
terraform init -backend-config=backend.hcl  # Configure backend

# Planning
terraform plan                    # Preview changes
terraform plan -out=tfplan        # Save plan
terraform plan -destroy           # Preview destroy

# Applying
terraform apply                   # Apply changes
terraform apply tfplan            # Apply saved plan
terraform apply -auto-approve     # Skip confirmation
terraform apply -target=aws_instance.web  # Apply specific resource

# Destroying
terraform destroy                 # Destroy all
terraform destroy -target=aws_instance.web  # Destroy specific

# State Management
terraform state list              # List resources
terraform state show aws_instance.web  # Show resource
terraform state rm aws_instance.web    # Remove from state
terraform state mv SOURCE DEST    # Rename resource
terraform import aws_instance.web i-123  # Import existing
terraform refresh                 # Sync state with reality

# Outputs
terraform output                  # Show all outputs
terraform output instance_ip      # Show specific output

# Formatting
terraform fmt                     # Format code
terraform fmt -check              # Check formatting
terraform fmt -recursive          # Format all files

# Validation
terraform validate                # Validate syntax
terraform graph                   # Generate dependency graph

# Workspace
terraform workspace list          # List workspaces
terraform workspace new dev       # Create workspace
terraform workspace select prod   # Switch workspace
```

---

## CloudFormation Commands Cheat Sheet

```bash
# Create Stack
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=KeyName,ParameterValue=value \
  --capabilities CAPABILITY_IAM

# Update Stack with Change Set
aws cloudformation create-change-set \
  --stack-name my-stack \
  --template-body file://template-v2.yaml \
  --change-set-name my-changes

aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name my-changes

aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changes

# Delete Stack
aws cloudformation delete-stack \
  --stack-name my-stack

# Stack Operations
aws cloudformation describe-stacks \
  --stack-name my-stack

aws cloudformation list-stacks

aws cloudformation get-template \
  --stack-name my-stack

# Drift Detection
aws cloudformation detect-stack-drift \
  --stack-name my-stack

aws cloudformation describe-stack-resource-drifts \
  --stack-name my-stack

# Validation
aws cloudformation validate-template \
  --template-body file://template.yaml

# StackSets
aws cloudformation create-stack-set \
  --stack-set-name my-stackset \
  --template-body file://template.yaml

aws cloudformation create-stack-instances \
  --stack-set-name my-stackset \
  --accounts 123456789012 234567890123 \
  --regions us-east-1 us-west-2
```

---

## Terraform Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/eks/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    
    # For multi-account setup
    role_arn       = "arn:aws:iam::123456789012:role/TerraformBackend"
  }
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  required_version = ">= 1.5.0"
}
```

---

## Common Terraform Patterns

### VPC with 3 AZs

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  enable_dns_hostnames = true
  enable_dns_support   = true
}

resource "aws_subnet" "private" {
  count  = 3
  vpc_id = aws_vpc.main.id
  
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "private-${count.index + 1}"
  }
}
```

### Security Group

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### EKS Cluster

```hcl
resource "aws_eks_cluster" "main" {
  name     = "prod-eks"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.27"

  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true
    endpoint_public_access  = true
  }

  enabled_cluster_log_types = ["api", "audit"]
}
```

---

## CloudFormation Template Patterns

### VPC with 3 AZs

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
  
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: private-1
  
  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: private-2
```

### Security Group

```yaml
Resources:
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server security group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
```

---

## Interview Quick Answers

**Q: "Terraform vs CloudFormation - when do you use each?"**
A: "Terraform for multi-cloud, better syntax, community modules. CloudFormation for AWS-only, day-0 service support, AWS-managed state, StackSets multi-account."

**Q: "What is Terraform state?"**
A: "JSON file mapping config to real resources. Remote state in S3 with DynamoDB locking for teams."

**Q: "How do you handle secrets?"**
A: "Never in code or state. Use AWS Secrets Manager, reference via data sources. Mark outputs sensitive."

**Q: "What are Terraform modules?"**
A: "Reusable collections of resources. Define once, use with different parameters across environments."

**Q: "What are CloudFormation change sets?"**
A: "Preview stack updates before applying. Shows exactly what will change. Execute if safe, delete if risky."

**Q: "How do you manage multi-environment?"**
A: "Terraform: Separate tfvars per environment or Terragrunt. CloudFormation: Parameters and nested stacks."

**Q: "Explain Terraform backend?"**
A: "S3 stores state, DynamoDB provides locking. Enables team collaboration and prevents concurrent modifications."

**Q: "What happens if Terraform state is lost?"**
A: "Disaster. Can't manage resources anymore. Must import all resources manually or restore from S3 version history."

**Q: "CloudFormation automatic rollback?"**
A: "Yes, on failure CloudFormation deletes partial resources and returns to previous state."

**Q: "Terraform count vs for_each?"**
A: "count uses numeric indexes (0,1,2). for_each uses keys. for_each better - no recreate when list changes."

---

## Troubleshooting Checklist

### Terraform Issues

- [ ] Run `terraform fmt` - Fix formatting
- [ ] Run `terraform validate` - Check syntax
- [ ] Check state lock in DynamoDB
- [ ] Verify AWS credentials: `aws sts get-caller-identity`
- [ ] Check provider version compatibility
- [ ] Review plan output carefully
- [ ] Check resource dependencies

### CloudFormation Issues

- [ ] Validate template: `aws cloudformation validate-template`
- [ ] Check stack events for error details
- [ ] Verify IAM permissions (CAPABILITY_IAM)
- [ ] Check resource limits (VPC limit, EIP limit)
- [ ] Review dependencies (DependsOn)
- [ ] Check parameter constraints
- [ ] Use change sets to preview updates

---

## Top 10 Critical Facts

1. **Terraform uses HCL**; CloudFormation uses JSON/YAML
2. **Terraform state** must be managed (S3 + DynamoDB); CloudFormation state is AWS-managed
3. **Terraform modules** are reusable; CloudFormation uses nested stacks
4. **CloudFormation** supports all AWS services immediately; Terraform waits for provider
5. **Change sets** in CloudFormation preview updates; terraform plan does same
6. **State locking** prevents concurrent modifications (critical for teams)
7. **Terraform import** brings existing resources under management
8. **CloudFormation StackSets** deploy across multiple accounts/regions
9. **Module versioning** critical for stability (pin versions)
10. **Both tools** are declarative (describe desired state, not steps)

---

## Quick Decision Guide

```
Need multi-cloud? → Terraform
AWS-only forever? → CloudFormation OR Terraform
Want cleaner syntax? → Terraform
Need day-0 AWS service support? → CloudFormation
Want rich module ecosystem? → Terraform
Using AWS Control Tower? → CloudFormation
Team knows Terraform? → Terraform
Team knows CloudFormation? → CloudFormation
Need StackSets (multi-account)? → CloudFormation
Want to manage state yourself? → Terraform (more control)
Want AWS to manage state? → CloudFormation (simpler)
```

**Most common answer:** Start with Terraform (flexibility), use CloudFormation when needed.

