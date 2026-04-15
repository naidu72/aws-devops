# Terraform & CloudFormation - Infrastructure as Code Interview Preparation

**Your Interview Focus:**
- Terraform fundamentals and advanced patterns
- CloudFormation for AWS-native infrastructure
- Terraform vs CloudFormation comparison
- Multi-environment management (DEV, QA, Staging, Production)
- State management and backends
- Terragrunt for DRY configurations
- Infrastructure automation and best practices

---

## 📁 File Structure

This Terraform & CloudFormation prep package contains 10 comprehensive guides:

### 1. **01-terraform-fundamentals.md** - Terraform Basics
**Duration:** 20-25 minutes

Core Terraform concepts:
- HCL syntax and structure
- Providers and resources
- State management (local vs remote)
- Backends (S3 + DynamoDB)
- Variables and outputs
- Data sources
- Lifecycle management
- **Your elevator pitch about Terraform**

### 2. **02-cloudformation-fundamentals.md** - CloudFormation Basics  
**Duration:** 20-25 minutes

AWS-native infrastructure as code:
- CloudFormation template structure (JSON/YAML)
- Resources, parameters, outputs
- Intrinsic functions (Ref, GetAtt, Sub)
- Stack management and updates
- Change sets and rollback
- StackSets for multi-account
- Nested stacks and modularity

### 3. **03-terraform-vs-cloudformation.md** - Comparison & When to Use
**Duration:** 15-20 minutes

Direct comparison:
- Terraform strengths vs CloudFormation strengths
- Multi-cloud vs AWS-only
- State management differences
- Community vs AWS support
- When to choose each tool
- Using both together

### 4. **04-terraform-modules.md** - Reusable Infrastructure
**Duration:** 20-25 minutes

Module design patterns:
- Module structure and best practices
- Input variables and outputs
- Module versioning
- Public vs private modules
- Module composition
- Testing modules

### 5. **05-terragrunt-patterns.md** - DRY Multi-Environment
**Duration:** 20-25 minutes

Terragrunt for environment management:
- DRY principle implementation
- Multi-account/multi-region setup
- Remote state configuration
- Dependency management
- Best practices

### 6. **06-eks-infrastructure.md** - EKS with Terraform & CloudFormation
**Duration:** 25-30 minutes

Complete EKS infrastructure:
- Terraform approach for EKS
- CloudFormation approach for EKS
- VPC networking
- Node groups and auto-scaling
- IAM roles and policies
- Add-ons deployment

### 7. **07-hands-on-labs.md** - Practical Exercises
**Duration:** 90-120 minutes total

5 complete lab exercises:
1. Build reusable Terraform modules for VPC
2. Deploy EKS cluster with CloudFormation
3. Set up Terragrunt multi-environment
4. Migrate Terraform state to S3
5. Implement infrastructure testing

### 8. **08-best-practices.md** - Production Standards
**Duration:** 15-20 minutes

Enterprise practices:
- Naming conventions
- Tagging strategies
- Security best practices
- Cost optimization
- Disaster recovery
- Compliance and governance

### 9. **09-interview-qa.md** - Interview Questions
**Duration:** 30-40 minutes

50+ questions with detailed answers:
- Terraform architecture
- CloudFormation design
- State management scenarios
- Module design
- Multi-environment patterns
- Troubleshooting

### 10. **10-quick-reference.md** - Cheat Sheet
**Duration:** 3-5 minutes

Fast lookup:
- Terraform commands
- CloudFormation CLI commands
- Common patterns
- Troubleshooting checklist
- Interview quick answers

---

## 🎯 How to Use These Files

### **First Time Studying (3-4 hours)**
1. Read `01-terraform-fundamentals.md` (understand Terraform)
2. Read `02-cloudformation-fundamentals.md` (understand CloudFormation)
3. Read `03-terraform-vs-cloudformation.md` (compare both)
4. Read `04-terraform-modules.md` (reusability)
5. Read `05-terragrunt-patterns.md` (multi-env)

### **Hands-on Learning (3-4 hours)**
1. Complete `07-hands-on-labs.md` exercises 1-3
2. Complete `07-hands-on-labs.md` exercises 4-5
3. Practice with your AWS account

### **Before Interview (45-60 minutes)**
1. Read `10-quick-reference.md` (key concepts)
2. Review `09-interview-qa.md` top 15 questions
3. Practice explaining Terraform vs CloudFormation
4. Draw infrastructure diagrams

---

## 🔥 Most Important Concepts

### **🔴 Critical (Must Know):**
1. Terraform state management (local vs remote)
2. CloudFormation stack operations
3. When to use Terraform vs CloudFormation
4. Module design patterns
5. Multi-environment strategies

### **🟡 Important:**
6. Terragrunt for DRY configurations
7. Remote backends (S3 + DynamoDB)
8. CloudFormation change sets
9. Import existing resources
10. State locking

### **🟢 Nice to Know:**
11. Terraform workspaces
12. CloudFormation StackSets
13. Custom providers
14. Testing strategies
15. Cost optimization

---

## 💡 Pro Tips for Interview

### Show Deep Understanding
❌ Don't: "I use Terraform for infrastructure"
✅ Do: "I use Terraform with remote S3 backend and DynamoDB locking for team collaboration. We organize code into reusable modules versioned in separate repos. For AWS-specific services without good Terraform support, we use CloudFormation, and import outputs using data sources."

### Explain Trade-offs
❌ Don't: "Terraform is better than CloudFormation"
✅ Do: "Terraform is multi-cloud and has excellent module ecosystem, but CloudFormation is AWS-native with day-0 support for new services. We use Terraform for core infrastructure (VPC, EKS) because of modules and multi-cloud possibility, but CloudFormation for AWS-specific features like Service Catalog."

### Reference Real Experience
❌ Don't: "We have Terraform files"
✅ Do: "We manage 4 environments (DEV, QA, Staging, Prod) using Terragrunt with shared modules. Each environment has its own tfvars file and remote state. We use Terraform for VPC (10.0.0.0/16 in prod, 10.1.0.0/16 in dev), EKS clusters, and IAM roles. State is in S3 with DynamoDB locking to prevent concurrent modifications."

---

## 🚀 Your Elevator Pitch (60 seconds)

"I manage infrastructure as code using both Terraform and CloudFormation. For multi-cloud capability and reusable modules, I use Terraform with remote S3 backend and state locking. I've built reusable modules for VPC, EKS, and security groups that are shared across 4 environments. For AWS-specific services or when we need AWS-native support, I use CloudFormation. I use Terragrunt to keep configurations DRY across dev, QA, staging, and production - sharing the same module code with environment-specific variables. All infrastructure changes are version-controlled in Git, peer-reviewed, and applied via CI/CD with automated terraform plan in pull requests."

---

## 📚 Study Schedule

**Beginner (First time):**
- Day 1: Files 01 + 02 (Terraform + CloudFormation basics)
- Day 2: Files 03 + 04 + 05 (Comparison, modules, Terragrunt)
- Day 3: Files 06 + 07 (EKS infra + hands-on labs)
- Day 4: Files 08 + 09 + 10 (Best practices, Q&A, quick ref)

**Intermediate (Refreshing):**
- Hour 1: Skim 01 + 02
- Hour 1: Read 03 (Terraform vs CloudFormation)
- Hour 1: Complete 2-3 labs
- Hour 1: Files 09 + 10 (Q&A + quick ref)

**Day-before-interview:**
- 30 min: Read file 10 (quick reference)
- 20 min: Review top 10 questions from file 09
- 20 min: Practice terraform commands
- Get rest!

---

## ✅ Self-Check Before Interview

- [ ] I understand Terraform state management (local vs remote)
- [ ] I know when to use Terraform vs CloudFormation
- [ ] I can design reusable Terraform modules
- [ ] I understand remote backends (S3 + DynamoDB)
- [ ] I know Terragrunt for multi-environment
- [ ] I can explain CloudFormation stacks and change sets
- [ ] I understand import and state manipulation
- [ ] I know tagging and governance strategies
- [ ] I can troubleshoot terraform plan/apply failures
- [ ] I understand the terraform vs cloudformation trade-offs

---

## 🎓 Good Luck!

Infrastructure as Code is critical to modern DevOps. Master both Terraform and CloudFormation to show versatility. Think about "why" you choose each tool for different scenarios.

**Remember:**
- Explain Terraform vs CloudFormation trade-offs
- Mention state management and locking
- Talk about multi-environment patterns
- Show module reusability thinking
- Reference your actual setup

**Go get that Senior DevOps role! 💪**

---

*Terraform & CloudFormation Interview Prep Created: April 2026*
*Focus: Terraform, CloudFormation, Terragrunt, multi-environment, EKS infrastructure*
