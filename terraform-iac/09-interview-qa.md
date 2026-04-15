# 09 - Terraform & CloudFormation Interview Questions & Answers

## Architecture & Fundamentals

### Q1: "What is Terraform and how does it work?"

**Answer (2-3 minutes):**
"Terraform is an open-source Infrastructure as Code tool by HashiCorp that lets you define and provision infrastructure using declarative configuration files written in HCL. Here's how it works:

1. **Write Configuration:** You define desired infrastructure in .tf files using HCL syntax
2. **Initialize:** `terraform init` downloads required providers (AWS, Azure, etc.)
3. **Plan:** `terraform plan` compares your configuration to the current state and shows what will change
4. **Apply:** `terraform apply` makes API calls to providers to create/update/delete resources
5. **State:** Terraform tracks resources in a state file (terraform.tfstate)

Key components:
- **Providers:** Plugins that interact with cloud APIs (AWS provider, Azure provider)
- **Resources:** Infrastructure components you want to create (EC2, VPC, S3)
- **State:** JSON file mapping your config to real resources
- **Modules:** Reusable collections of resources

For teams, we use remote state in S3 with DynamoDB locking so multiple people can collaborate without conflicts. Terraform's key advantage is it's declarative - you describe what you want, not how to create it."

---

### Q2: "What is AWS CloudFormation and how is it different from Terraform?"

**Answer (2-3 minutes):**
"CloudFormation is AWS's native Infrastructure as Code service. You define AWS resources in JSON or YAML templates, and CloudFormation provisions them. Key differences from Terraform:

**CloudFormation:**
- AWS-only (but supports all AWS services immediately on release day)
- Templates in JSON/YAML (more verbose than Terraform's HCL)
- AWS manages state automatically (you don't see or manage it)
- Deep integration with AWS services (Service Catalog, Control Tower, StackSets)
- Free to use
- Nested stacks for modularity

**Terraform:**
- Multi-cloud (AWS, Azure, GCP, 3000+ providers)
- HCL syntax (cleaner, more readable)
- You manage state (typically S3 + DynamoDB for teams)
- Rich community module ecosystem
- New AWS services lag behind CloudFormation (wait for provider update)
- Open source, free

**When I choose CloudFormation:** AWS-only infrastructure, need day-0 support for new services, using AWS Control Tower or StackSets for multi-account.

**When I choose Terraform:** Multi-cloud scenarios, prefer cleaner syntax, want community modules, need flexibility to move clouds in future.

In some projects, I use both: Terraform for core infrastructure (VPC, EKS) and CloudFormation for organization-wide governance via StackSets."

---

## State Management

### Q3: "Explain Terraform state and why it's important"

**Answer (2-3 minutes):**
"Terraform state is a JSON file that acts as Terraform's source of truth, mapping your configuration to real infrastructure resources. It's critical because:

1. **Resource Tracking:** State stores resource IDs (vpc-abc123, i-def456) so Terraform knows what it manages
2. **Change Detection:** Terraform compares desired state (your config) vs current state to determine what to do
3. **Metadata:** Stores resource dependencies and attributes needed for other resources
4. **Performance:** Avoids API calls to fetch every resource attribute

**Local State (default):**
- terraform.tfstate file in current directory
- OK for solo developer, learning, throwaway environments
- Problems: no collaboration, no locking, risk of data loss

**Remote State (production):**
```hcl
terraform {
  backend \"s3\" {
    bucket         = \"company-terraform-state\"
    key            = \"prod/eks/terraform.tfstate\"
    region         = \"us-east-1\"
    encrypt        = true
    dynamodb_table = \"terraform-locks\"
  }
}
```

Benefits:
- Team collaboration (shared state in S3)
- State locking via DynamoDB (prevents concurrent modifications)
- Versioning (S3 versioning enabled for rollback)
- Encryption at rest
- Disaster recovery (backup)

**Critical:** Never manually edit state. Use terraform state commands. If state is lost, you can't manage resources and must import everything manually."

---

### Q4: "How does CloudFormation handle state?"

**Answer (1-2 minutes):**
"CloudFormation state is AWS-managed - you don't see or manage it directly. When you create a stack, CloudFormation stores:
- Template used
- Parameters provided
- Resources created (logical IDs → physical IDs)
- Outputs
- Stack metadata

Advantages:
- No S3 bucket to set up or manage
- No state locking concerns (AWS handles it)
- Automatic versioning
- Automatic backup
- Simpler operations (one less thing to manage)

You interact with state via CloudFormation APIs:
- `describe-stacks` - See stack resources
- `get-template` - Get template used
- `detect-stack-drift` - Find manual changes

Unlike Terraform where state loss is catastrophic, CloudFormation state is always safe in AWS. However, you have less control - can't manipulate state directly like with Terraform state commands."

---

## Modules & Reusability

### Q5: "How do you design reusable Terraform modules?"

**Answer (2-3 minutes):**
"I design Terraform modules to be reusable, configurable, and composable. Here's my approach:

**Structure:**
```
modules/vpc/
├── main.tf         # Resources
├── variables.tf    # Inputs
├── outputs.tf      # Outputs
├── README.md       # Documentation
└── examples/       # Usage examples
```

**Principles:**
1. **Single Responsibility:** VPC module creates VPC only, not EKS or RDS
2. **Configurable:** Important parameters exposed as variables with sensible defaults
3. **Outputs:** Return useful values (IDs, ARNs) for other modules
4. **No Hard-coding:** Everything via variables (no hard-coded account IDs, regions)
5. **Versioned:** Git tags for versions (v1.0.0, v1.1.0)

**Example VPC Module:**
```hcl
# Usage
module \"vpc_prod\" {
  source = \"git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0\"
  
  name               = \"prod-vpc\"
  vpc_cidr           = \"10.0.0.0/16\"
  availability_zones = [\"us-east-1a\", \"us-east-1b\", \"us-east-1c\"]
  
  tags = {
    Environment = \"production\"
  }
}

# Consume outputs
resource \"aws_eks_cluster\" \"prod\" {
  vpc_config {
    subnet_ids = module.vpc_prod.private_subnet_ids
  }
}
```

**Versioning:** Pin module versions to prevent breaking changes:
```hcl
version = \"~> 1.2\"  # Allow 1.x updates, not 2.x
```

**Testing:** terraform validate, terraform plan in test environment, example configurations in examples/ directory."

---

## Multi-Environment Management

### Q6: "How do you manage multiple environments (dev, qa, staging, production) with Terraform?"

**Answer (2-3 minutes):**
"I use separate tfvars files per environment, with the same root module code. Directory structure:

```
terraform/
├── main.tf               # Root module (same for all envs)
├── variables.tf
├── outputs.tf
├── backend.tf
├── environments/
│   ├── dev.tfvars        # Dev parameters
│   ├── qa.tfvars         # QA parameters
│   ├── staging.tfvars    # Staging parameters
│   └── production.tfvars # Production parameters
```

**Example tfvars:**
```hcl
# dev.tfvars
environment      = \"dev\"
vpc_cidr         = \"10.1.0.0/16\"
instance_type    = \"t3.micro\"
eks_node_count   = 2

# production.tfvars
environment      = \"production\"
vpc_cidr         = \"10.0.0.0/16\"
instance_type    = \"c5.2xlarge\"
eks_node_count   = 12
```

**Deployment:**
```bash
terraform apply -var-file=environments/dev.tfvars
terraform apply -var-file=environments/production.tfvars
```

**Remote State Per Environment:**
```hcl
# Each env has separate state file
backend \"s3\" {
  bucket = \"company-terraform-state\"
  key    = \"dev/eks/terraform.tfstate\"     # dev
  # key    = \"prod/eks/terraform.tfstate\"   # prod
}
```

**Alternative: Terragrunt** (for very complex multi-account setups):
- DRY principle (Don't Repeat Yourself)
- Shared configuration
- Dependency management
- Better for 10+ environments or multi-account AWS

**Key principle:** Same code, different parameters. Changes tested in dev, promoted through qa/staging to production."

---

### Q7: "What is Terragrunt and when would you use it?"

**Answer (1-2 minutes):**
"Terragrunt is a thin wrapper around Terraform that helps keep configurations DRY (Don't Repeat Yourself). It's useful for managing multiple environments or accounts with shared configuration.

**Problem without Terragrunt:**
```
terraform/
├── dev/
│   ├── backend.tf (duplicated)
│   ├── provider.tf (duplicated)
│   └── main.tf
├── qa/
│   ├── backend.tf (duplicated)
│   ├── provider.tf (duplicated)
│   └── main.tf
└── prod/
    ├── backend.tf (duplicated)
    ├── provider.tf (duplicated)
    └── main.tf
```

**Solution with Terragrunt:**
```
terragrunt/
├── terragrunt.hcl (shared config)
├── modules/ (Terraform code)
├── dev/
│   └── terragrunt.hcl (dev-specific)
├── qa/
│   └── terragrunt.hcl (qa-specific)
└── prod/
    └── terragrunt.hcl (prod-specific)
```

**When to use:**
- 5+ environments or AWS accounts
- Shared remote state configuration
- Need dependency management between modules
- Multi-account AWS setups

**When NOT to use:**
- Simple setups (2-3 environments) - tfvars is simpler
- Team unfamiliar with Terragrunt - adds complexity"

---

## Advanced Scenarios

### Q8: "How do you import existing infrastructure into Terraform?"

**Answer (2 minutes):**
"Importing brings existing resources under Terraform management without recreating them. Process:

1. **Write Configuration:** Define the resource in .tf file
```hcl
resource \"aws_instance\" \"web\" {
  ami           = \"ami-12345\"
  instance_type = \"t3.micro\"
  # ... all other attributes
}
```

2. **Import:** Link resource to state
```bash
terraform import aws_instance.web i-1234567890abcdef0
```

3. **Verify:** Run terraform plan (should show no changes)
4. **Refine:** Adjust config until plan shows zero changes

**Challenges:**
- Must manually write configuration (Terraform doesn't generate it)
- Need to match ALL attributes (tags, security groups, etc.)
- Time-consuming for many resources

**Tools to help:**
- `terraform show` after import (shows state)
- `terraformer` tool (automates import + config generation)
- `aws` CLI to inspect resource details

**Use case:** Taking over infrastructure created manually or via CloudFormation."

---

### Q9: "Explain CloudFormation change sets and why they're important"

**Answer (2 minutes):**
"Change sets let you preview CloudFormation stack updates before applying them. It's like `terraform plan` but for CloudFormation.

**Process:**
```bash
# 1. Create change set
aws cloudformation create-change-set \\
  --stack-name prod-eks \\
  --template-body file://updated-template.yaml \\
  --change-set-name update-node-count

# 2. Review changes
aws cloudformation describe-change-set \\
  --stack-name prod-eks \\
  --change-set-name update-node-count

# Output shows:
# Action: Modify
# ResourceType: AWS::EKS::Nodegroup
# Details:
#   - Property: DesiredSize
#     Before: 6
#     After: 12
#   - Replacement: False

# 3. Execute if safe
aws cloudformation execute-change-set \\
  --change-set-name update-node-count

# OR delete if risky
aws cloudformation delete-change-set \\
  --change-set-name update-node-count
```

**Why critical:**
- Prevents surprises (know exactly what will change)
- Shows if resources will be replaced (downtime)
- Safe production updates (preview first)
- Can review with team before executing

**Always use change sets for production updates!**"

---

## Security & Best Practices

### Q10: "How do you handle secrets in Terraform?"

**Answer (2 minutes):**
"Never store secrets in Terraform files or state. Approach:

**1. Use AWS Secrets Manager or Parameter Store:**
```hcl
data \"aws_secretsmanager_secret_version\" \"db_password\" {
  secret_id = \"prod/db/password\"
}

resource \"aws_db_instance\" \"main\" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # Secret never in code or state as plaintext
}
```

**2. Sensitive Outputs:**
```hcl
output \"database_password\" {
  value     = aws_db_instance.main.password
  sensitive = true  # Not displayed in terminal
}
```

**3. Environment Variables:**
```bash
export TF_VAR_db_password=$(aws secretsmanager get-secret-value ...)
```

**4. State Encryption:**
```hcl
backend \"s3\" {
  bucket  = \"terraform-state\"
  encrypt = true  # Encrypt state at rest
}
```

**5. IAM Roles (not access keys):**
```hcl
provider \"aws\" {
  # Uses IAM role from EC2 instance profile
  # No access keys in code
}
```

**Never:**
- ❌ Hard-code passwords in .tf files
- ❌ Commit secrets to Git
- ❌ Pass secrets via terraform.tfvars (tracked in Git)
- ❌ Log secrets in CI/CD output"

---

## Top 10 Quick Interview Answers

### Q11: "Terraform vs CloudFormation in one sentence"
"Terraform is multi-cloud with better syntax and modules; CloudFormation is AWS-native with day-0 service support and AWS-managed state."

### Q12: "What is Terraform state in one sentence?"
"JSON file mapping Terraform configuration to real infrastructure resource IDs and attributes."

### Q13: "What are Terraform modules?"
"Reusable collections of Terraform resources defined once and used with different parameters across environments."

### Q14: "How do you version Terraform modules?"
"Git tags with semantic versioning (v1.0.0, v1.1.0), pinned in module source with ?ref=v1.2.0."

### Q15: "What is a CloudFormation change set?"
"Preview of stack updates showing exactly what resources will change before executing the update."

### Q16: "How do you manage multi-environment in Terraform?"
"Separate tfvars files per environment (dev.tfvars, prod.tfvars) with the same root module code."

### Q17: "What happens if Terraform state is lost?"
"Disaster - can't manage resources anymore. Must import all resources manually or restore from S3 versions."

### Q18: "CloudFormation automatic rollback?"
"Yes, on create/update failure CloudFormation automatically deletes partial resources and returns to previous state."

### Q19: "When would you use CloudFormation StackSets?"
"Deploy same template across multiple AWS accounts and regions from single operation for organization-wide standards."

### Q20: "Terraform count vs for_each?"
"count uses numeric indexes, for_each uses keys from map/set. for_each better - no resource recreation when list changes."

