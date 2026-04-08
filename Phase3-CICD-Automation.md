# Phase 3: CI/CD & Automation

## Study Duration: Days 3-4
## Topics Covered: CI/CD Fundamentals, CodePipeline/CodeBuild/CodeDeploy, IaC

---

## 3.1 CI/CD Fundamentals & Pipeline Design

### CI/CD Concepts

#### 1. Continuous Integration (CI)

**Definition:** Automatically build and test code on every commit

**Benefits:**
- Catch bugs early
- Reduce integration issues
- Fast feedback loop
- Maintain code quality
- Reduce manual testing

**Pipeline Stages:**
1. **Trigger:** Code committed/PR opened
2. **Build:** Compile, run tests, static analysis
3. **Test:** Unit, integration, security tests
4. **Artifact:** Package application
5. **Notify:** Status to developers

**Example CI Flow:**
```
Developer commits → Webhook → CodeBuild
                               ↓
                        Run tests
                               ↓
                        Tests pass? → Store artifact (S3)
                                           ↓
                                    Notify developer ✓
                               ✗ → Notify developer ✗
```

#### 2. Continuous Deployment/Delivery (CD)

**Continuous Delivery:** Automatically prepare for production release
**Continuous Deployment:** Automatically deploy to production

**Deployment Strategies:**

1. **Blue-Green:**
   - Maintain two identical environments (Blue/Green)
   - Traffic switched from Blue to Green
   - Quick rollback (switch back to Blue)
   - Double resource cost temporarily
   - Zero-downtime deployment

```
Current Production (Blue)
↓ (ALB)
New Version (Green)
↓ (When ready, traffic switches)
New Production (Green)
↓ (Old can be cleaned up)
```

2. **Canary:**
   - Route small % to new version (5%)
   - Monitor for issues
   - Gradually increase % (25%, 50%, 100%)
   - Fast rollback if issues
   - Reduced risk compared to blue-green

3. **Rolling:**
   - Gradually replace old instances
   - 1 instance at a time
   - No downtime (load balanced)
   - Slower rollout
   - Original strategy for deployments

#### 3. Pipeline Architecture Design

**Source Stage:**
- GitHub, CodeCommit, GitLab
- Branch filtering (only on main?)
- Webhook triggers vs scheduled

**Build Stage:**
- Compile code
- Run unit tests
- Static analysis (SonarQube)
- Build Docker image
- Push to registry

**Test Stage:**
- Integration tests
- Security scanning (SAST)
- Performance tests
- Infrastructure tests

**Deploy Stage:**
- Dev environment (auto)
- Staging environment (manual approval)
- Production (manual approval)

**Artifact Management:**
- CodeArtifact for libraries
- ECR for Docker images
- S3 for binaries/packages
- Versioning and retention policies

#### 4. Testing in CI/CD

**Unit Tests:**
- Fast, isolated
- Run on every commit
- No external dependencies
- 70-80% of tests

**Integration Tests:**
- Database, APIs, services
- Slower (takes seconds/minutes)
- Run on every commit (subset) or per merge
- 15-20% of tests

**End-to-End Tests:**
- Full application flow
- Slow (takes minutes)
- Run on releases or staging
- 5-10% of tests

**Test Coverage:**
- Aim for 80%+ code coverage
- Focus on critical paths
- Branch coverage matters
- Code complexity analysis

#### 5. Secrets Management in CI/CD

**Best Practices:**
- Never commit secrets to Git
- Use secret manager service
- Secret accessible only to build environment
- Audit access to secrets
- Rotate regularly
- Different secrets per environment

**Implementation:**
```bash
# In CI/CD (CodeBuild, GitHub Actions, etc.)
DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id prod/db/password \
  --query SecretString --output text)

# Use in application
./deploy.sh --db-password "$DB_PASSWORD"
```

### Interview Questions Deep Dive

**Q1: Design a CI/CD pipeline from commit to production**

*Expected Answer Structure:*
- Diagram or flow description
- Source: GitHub main branch
- Build: Compile, unit tests, Docker build
- Test: Integration tests, security scan
- Deploy to Dev (auto)
- Deploy to Staging (manual approval)
- Deploy to Prod (manual approval)
- Rollback capability
- Monitoring and alerting

**Q2: What's the difference between CI and CD?**

*Expected Answer:*
- CI: Automated build and test on every commit
- CD: Automated deployment to environments
- CI: Catch bugs early
- CD: Deliver faster to production
- CI: Happens for every commit
- CD: Happens per release/approval
- CI/CD together: Full automation pipeline

**Q3: How do you implement zero-downtime deployments?**

*Expected Answer:*
- Blue-green deployment with load balancer
- Health checks before switching traffic
- ALB/NLB routes traffic
- Warm up new instances
- Run smoke tests before switching
- Automatic rollback on failure
- Quick rollback capability

**Q4: Explain blue-green vs canary deployment strategies**

*Expected Answer:*
- **Blue-Green:** Two full environments, instant switch
  - Pros: Zero downtime, easy rollback
  - Cons: Double cost, all-or-nothing
  - Best for: Critical systems, known deployments
- **Canary:** Gradual traffic shift (5% → 100%)
  - Pros: Reduced risk, early detection
  - Cons: Longer rollout, complexity
  - Best for: New features, uncertain deployments

**Q5: How do you manage secrets securely in CI/CD pipelines?**

*Expected Answer:*
- Use Secrets Manager or Parameter Store
- Secrets injected at build time only
- Access only by build container
- Never log secrets
- Different secret per environment
- Rotate periodically
- Audit access in CloudTrail
- IAM policies restrict who can access

---

## 3.2 AWS CodePipeline, CodeBuild, CodeDeploy

### CodePipeline Architecture

#### 1. Pipeline Concept

**Pipeline = Stages + Actions**
- Stages execute in order
- Actions within stage can execute in parallel
- Manual approval gates
- Each action either succeeds or fails

**Example Pipeline:**
```
Source (GitHub)
    ↓
Build (CodeBuild)
    ↓
Test (CodeBuild)
    ↓
[Manual Approval]
    ↓
Deploy Dev (CodeDeploy)
    ↓
Deploy Staging (CodeDeploy)
    ↓
[Manual Approval]
    ↓
Deploy Prod (CodeDeploy)
```

#### 2. Source Stage

**Integrations:**
- CodeCommit (AWS Git)
- GitHub
- GitHub Enterprise
- GitLab
- Bitbucket
- S3

**Triggers:**
- Webhook (on push/PR)
- Polling (check periodically)
- CloudWatch Events (scheduled)

```bash
# Create pipeline with GitHub source
aws codepipeline create-pipeline --cli-input-json '{
  "pipeline": {
    "name": "my-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/CodePipelineRole",
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "GitHub",
            "actionTypeId": {
              "category": "Source",
              "owner": "ThirdParty",
              "provider": "GitHub",
              "version": "1"
            },
            "outputArtifacts": [
              {"name": "SourceOutput"}
            ],
            "configuration": {
              "Owner": "myorg",
              "Repo": "my-repo",
              "Branch": "main",
              "OAuthToken": "github-token"
            }
          }
        ]
      }
    ]
  }
}'
```

#### 3. CodeBuild Stage

**Build Environment:**
- Pre-built images (Node, Python, Java, Go, Ruby, .NET, Docker)
- Custom Docker images
- Compute types: build.general1.small → build.general1.3xlarge

**BuildSpec File (buildspec.yml):**
```yaml
version: 0.2

env:
  variables:
    AWS_REGION: us-east-1
  parameter-store:
    DB_PASSWORD: /prod/db/password
  secrets-manager:
    DOCKER_HUB_TOKEN: dockerhub:token

phases:
  pre_build:
    commands:
      - echo Logging in to Docker Hub
      - echo $DOCKER_HUB_TOKEN | docker login -u $DOCKER_USERNAME --password-stdin
      - AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - REPO_URL=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app
      - IMAGE_TAG=latest

  build:
    commands:
      - echo Building Docker image
      - docker build -t $REPO_URL:$IMAGE_TAG .
      - docker tag $REPO_URL:$IMAGE_TAG $REPO_URL:$CODEBUILD_BUILD_NUMBER

  post_build:
    commands:
      - echo Pushing image to ECR
      - docker push $REPO_URL:$IMAGE_TAG
      - docker push $REPO_URL:$CODEBUILD_BUILD_NUMBER
      - echo Writing image definitions file
      - printf '[{"name":"my-app","imageUri":"%s"}]' $REPO_URL:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
  name: BuildArtifact

cache:
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'

reports:
  UnitTests:
    files:
      - 'coverage/cobertura-coverage.xml'
    file-format: 'COBERTURAXML'
  SecurityScan:
    files:
      - 'security-report.json'
```

**Build Caching:**
- Cache Docker layers
- Cache dependencies (npm, pip, maven)
- Speeds up builds significantly

#### 4. CodeDeploy Stage

**Deployment Types:**

1. **In-Place:**
   - Stop application on instance
   - Deploy new version
   - Start application
   - Downtime during deployment
   - Simpler, cheaper

2. **Blue-Green:**
   - New instances (Green) with new version
   - Switch traffic from Blue → Green
   - Zero downtime
   - More resources needed
   - Easy rollback

3. **Canary:**
   - Route small % of traffic to new version
   - Monitor metrics
   - Gradually increase traffic
   - Rollback on issues

**AppSpec File (appspec.yaml):**
```yaml
version: 0.0

Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: !Ref TaskDefinition
        LoadBalancerInfo:
          ContainerName: my-app
          ContainerPort: 8080
        PlatformVersion: LATEST
        NetworkConfiguration:
          AwsvpcConfiguration:
            Subnets:
              - subnet-12345
            SecurityGroups:
              - sg-12345

Hooks:
  BeforeInstall:
    - Location: scripts/before-install.sh
      Timeout: 300
      RunAs: ec2-user
  
  AfterInstall:
    - Location: scripts/after-install.sh
      Timeout: 300
      RunAs: ec2-user
  
  BeforeAllowTraffic:
    - Location: scripts/health-check.sh
      Timeout: 60
      RunAs: ec2-user

  AfterAllowTraffic:
    - Location: scripts/after-traffic.sh
      Timeout: 300
      RunAs: ec2-user
```

**Lifecycle Hooks:**
- `BeforeBlockTraffic`: Before removing from LB
- `AfterBlockTraffic`: After removing from LB
- `BeforeAllowTraffic`: Before adding to LB
- `AfterAllowTraffic`: After adding to LB

#### 5. Pipeline Execution & Approval

**Manual Approval Action:**
```bash
# Add approval stage to pipeline
{
  "name": "Approval",
  "actions": [
    {
      "name": "ManualApproval",
      "actionTypeId": {
        "category": "Approval",
        "owner": "AWS",
        "provider": "Manual",
        "version": "1"
      },
      "configuration": {
        "CustomData": "Please review staging environment before production deployment",
        "NotificationArn": "arn:aws:sns:us-east-1:123456789012:prod-approvals"
      }
    }
  ]
}
```

**Execution Status:**
- In Progress
- Succeeded
- Failed
- Stopped

#### 6. AppConfig for Feature Flags

**Use Case:** Control feature rollout without redeployment

```bash
# Create configuration
aws appconfig create-application --name myapp
aws appconfig create-configuration-profile \
  --application-id app-id \
  --name feature-flags \
  --location-uri s3://mybucket/features.json

# Define feature flags
{
  "features": {
    "new_ui": {
      "enabled": true,
      "rollout_percentage": 25
    },
    "beta_api": {
      "enabled": false
    }
  }
}

# Application polls AppConfig for updates
aws appconfig get-configuration \
  --application myapp \
  --environment prod \
  --configuration feature-flags \
  --client-id my-client
```

### Interview Questions Deep Dive

**Q1: How would you implement automated testing in CodeBuild?**

*Expected Answer:*
- Multiple build phases for different test types
- Unit tests: fast, run first
- Integration tests: slower, run after unit tests
- Security tests: static analysis, dependency check
- Fail build if tests fail
- Generate reports and artifacts
- Store coverage reports for trend analysis

**Q2: Explain CodeDeploy lifecycle hooks and their use cases**

*Expected Answer:*
- `BeforeBlockTraffic`: Pre-drain tasks (graceful shutdown)
- `AfterBlockTraffic`: Validation before removal
- `BeforeAllowTraffic`: Health checks before adding
- `AfterAllowTraffic`: Post-deployment tasks (warmup)
- Use for: Graceful shutdown, health validation, metrics collection
- Critical for zero-downtime deployments

**Q3: How do you handle failures in CodePipeline and implement rollbacks?**

*Expected Answer:*
- Pipeline stops on action failure
- Rollback options depend on deployment type
- Blue-green: Switch traffic back to blue
- In-place: Redeploy previous version
- Automatic rollback on health check failure
- Manual rollback if needed
- Keep previous versions/artifacts for rollback
- Test rollback procedures

**Q4: Design a multi-environment CI/CD pipeline (dev, staging, prod)**

*Expected Answer:*
- Single source, multiple deploy stages
- Dev: Auto-deploy on main branch
- Staging: Manual approval, staging env vars
- Prod: Manual approval, prod secrets
- Different AppSpecs/task definitions per environment
- Different health check endpoints per environment
- Cross-stack template or environment parameter substitution

---

## 3.3 Infrastructure as Code (IaC)

### CloudFormation Fundamentals

#### 1. Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'VPC with EC2 instances and RDS'

Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]
  
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues: [t2.micro, t2.small, t2.medium]

Mappings:
  RegionAMI:
    us-east-1:
      AMI: ami-0c55b159cbfafe1f0
    us-west-2:
      AMI: ami-003eba3eddb97e333

Conditions:
  IsProduction: !Equals [!Ref EnvironmentName, 'prod']
  CreateRDSMultiAZ: !Or
    - !Condition IsProduction
    - !Equals [!Ref EnvironmentName, 'staging']

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']

  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP/HTTPS
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${EnvironmentName}-VPC-ID'
  
  SubnetId:
    Description: Public Subnet ID
    Value: !Ref PublicSubnet
```

#### 2. CloudFormation Concepts

**Stacks:**
- Unit of deployment
- Contains all resources
- Created, updated, deleted together

**Change Sets:**
- Preview changes before applying
- Safe way to update stacks
- Shows what will be created/deleted/modified

```bash
# Create change set
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changes \
  --template-body file://template.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=prod

# Review changes
aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name my-changes

# Execute change set
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changes
```

**Drift Detection:**
- Detects manual changes to resources
- Warns about non-IaC modifications

```bash
# Detect drift
aws cloudformation detect-stack-drift --stack-name my-stack

# View drift details
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-stack
```

#### 3. Terraform Fundamentals

**HCL Syntax:**
```hcl
provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  default = "us-east-1"
  type    = string
}

variable "environment" {
  default = "dev"
  type    = string
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "${var.environment}-public-subnet"
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
}
```

#### 4. Terraform State Management

**Local State (Development):**
```
terraform.tfstate (contains all resource info)
terraform.tfstate.backup
.gitignore: terraform.tfstate
```

**Remote State (Team/Production):**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

**State Locking:**
- Prevents concurrent modifications
- DynamoDB table with terraform lock entries
- Automatic locking/unlocking

**State Security:**
- Enable S3 encryption
- Enable versioning
- Restrict access (IAM policies)
- Never commit to Git
- Use .gitignore

#### 5. Terraform Modules

**Module Structure:**
```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
├── rds/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
└── root/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

**Module Usage:**
```hcl
module "vpc" {
  source = "./modules/vpc"

  environment = var.environment
  cidr_block  = "10.0.0.0/16"
}

module "rds" {
  source = "./modules/rds"

  vpc_id                = module.vpc.vpc_id
  db_subnet_group_name  = module.vpc.db_subnet_group_name
  environment           = var.environment
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

### Interview Questions Deep Dive

**Q1: What's the difference between CloudFormation and Terraform?**

*Expected Answer:*
- **CloudFormation:** AWS-native, JSON/YAML, deep AWS integration
- **Terraform:** Multi-cloud, HCL, open-source
- CloudFormation: Great for AWS-only projects, tight integration
- Terraform: Better for multi-cloud, more community modules
- CloudFormation: Native change sets, drift detection
- Terraform: Better module system, easier to learn
- CloudFormation: Supports all AWS services immediately
- Terraform: May lag on new features

**Q2: How do you manage Terraform state in a team environment?**

*Expected Answer:*
- Remote S3 backend with encryption
- State locking via DynamoDB
- Restrict S3 access via IAM
- Enable versioning for rollback
- Never commit tfstate to Git
- Use separate state per environment
- Backup state files
- Use terraform workspaces for multi-environment

**Q3: Explain drift detection and why it's important**

*Expected Answer:*
- Drift: Resources manually changed outside IaC
- Problem: IaC becomes source of truth problem
- Detection: CloudFormation can detect
- Why important: Know actual state vs desired
- Prevents configuration drift over time
- Remediation: Sync or revert manual changes
- Best practice: Don't allow manual changes

**Q4: Design an IaC strategy for multi-environment deployments**

*Expected Answer:*
- Single template/module, different parameters
- Separate state per environment (dev, staging, prod)
- Different values.yaml or tfvars per environment
- Parameter Store for env-specific configs
- Separate AWS accounts for prod isolation
- Automation: Infrastructure changes via CI/CD
- Change sets for review before apply
- Rollback procedures for failed deployments

---

## CI/CD Best Practices

1. **Fail Fast:** Quick feedback on breaking changes
2. **Automate Tests:** Unit, integration, security, performance
3. **Immutable Artifacts:** Docker images tagged by build
4. **Environment Parity:** Dev/staging/prod similar
5. **Secrets Management:** Never in code or logs
6. **Monitoring:** Health checks, metrics, alerts
7. **Rollback Capability:** Quick revert if issues
8. **Documentation:** Pipeline behavior and runbooks
9. **Access Control:** Who can approve deployments
10. **Cost Optimization:** Clean up old artifacts/images

---

## Architecture Decision Record (ADR) Template

Use this for design decisions:

```markdown
# ADR: Use Blue-Green Deployment for Production

## Status: Accepted

## Context
Need zero-downtime deployments to production.

## Decision
Implement blue-green deployment strategy using AWS CodeDeploy.

## Consequences
- Positive: Zero downtime, quick rollback
- Negative: Double resource cost during deployment
- Consideration: Need load balancer configuration
```

---

**Total Phase 3 Study Time: 5-6 hours**
