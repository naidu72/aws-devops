# 02 - CloudFormation Fundamentals: AWS-Native Infrastructure as Code

## What is AWS CloudFormation?

CloudFormation is AWS's native Infrastructure as Code service. You define AWS resources in JSON or YAML templates, and CloudFormation provisions them automatically. It's deeply integrated with AWS and supports all AWS services on day-1 of release.

### CloudFormation vs Terraform (Quick Comparison)

| Aspect | CloudFormation | Terraform |
|--------|----------------|-----------|
| Provider | AWS only | Multi-cloud |
| Language | JSON/YAML | HCL |
| State | AWS managed | You manage (S3) |
| New Services | Day-0 support | Wait for provider |
| Modules | Nested stacks | Modules |
| Cost | Free | Free (open source) |
| Learning Curve | Steeper (verbose) | Gentler (cleaner syntax) |

---

## CloudFormation Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'EKS Cluster with VPC and Node Group'

Parameters:
  ClusterName:
    Type: String
    Default: prod-eks-cluster
    Description: Name of the EKS cluster
  
  NodeInstanceType:
    Type: String
    Default: t3.large
    AllowedValues:
      - t3.medium
      - t3.large
      - c5.xlarge
    Description: EC2 instance type for nodes

Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0c55b159cbfafe1f0
    us-west-2:
      AMI: ami-0d1cd67c26f5fca19

Conditions:
  IsProduction: !Equals [!Ref EnvironmentType, 'production']
  CreateMultiAZ: !Equals [!Ref EnvironmentType, 'production']

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
        SecurityGroupIds:
          - !Ref ClusterSecurityGroup
        EndpointPrivateAccess: true
        EndpointPublicAccess: true
      Logging:
        ClusterLogging:
          EnabledTypes:
            - Type: api
            - Type: audit
            - Type: authenticator
            - Type: controllerManager
            - Type: scheduler
      Tags:
        - Key: Name
          Value: !Ref ClusterName
        - Key: Environment
          Value: !Ref EnvironmentType
        - Key: ManagedBy
          Value: CloudFormation

  EKSClusterRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: eks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
        - arn:aws:iam::aws:policy/AmazonEKSVPCResourceController

  NodeGroup:
    Type: AWS::EKS::Nodegroup
    DependsOn: EKSCluster
    Properties:
      ClusterName: !Ref EKSCluster
      NodegroupName: !Sub '${ClusterName}-node-group'
      NodeRole: !GetAtt NodeInstanceRole.Arn
      Subnets:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      ScalingConfig:
        MinSize: 3
        MaxSize: 12
        DesiredSize: 6
      InstanceTypes:
        - !Ref NodeInstanceType
      AmiType: AL2_x86_64
      Tags:
        Name: !Sub '${ClusterName}-nodes'

Outputs:
  ClusterName:
    Description: EKS Cluster Name
    Value: !Ref EKSCluster
    Export:
      Name: !Sub '${AWS::StackName}-ClusterName'
  
  ClusterEndpoint:
    Description: EKS Cluster Endpoint
    Value: !GetAtt EKSCluster.Endpoint
    Export:
      Name: !Sub '${AWS::StackName}-ClusterEndpoint'
  
  ClusterArn:
    Description: EKS Cluster ARN
    Value: !GetAtt EKSCluster.Arn
```

---

## Intrinsic Functions

```yaml
# Ref - Reference parameter or resource
!Ref ClusterName
# Returns: "prod-eks-cluster"

# GetAtt - Get attribute of resource
!GetAtt EKSCluster.Endpoint
# Returns: "https://XXXXX.eks.amazonaws.com"

# Sub - String substitution
!Sub '${ClusterName}-node-group'
# Returns: "prod-eks-cluster-node-group"

# Join - Join list of strings
!Join ['-', ['eks', 'cluster', !Ref EnvironmentType]]
# Returns: "eks-cluster-production"

# Select - Select item from list
!Select [0, !GetAZs '']
# Returns: "us-east-1a"

# GetAZs - Get availability zones
!GetAZs ''
# Returns: ["us-east-1a", "us-east-1b", "us-east-1c"]

# If - Conditional
!If [IsProduction, 'c5.xlarge', 't3.medium']
# Returns: "c5.xlarge" if IsProduction=true, else "t3.medium"

# Equals - Comparison
!Equals [!Ref EnvironmentType, 'production']
# Returns: true or false

# Not - Logical NOT
!Not [!Equals [!Ref EnvironmentType, 'dev']]
```

---

## Stack Operations

### Create Stack

```bash
# Create stack from template
aws cloudformation create-stack \
  --stack-name prod-eks-cluster \
  --template-body file://eks-cluster.yaml \
  --parameters \
    ParameterKey=ClusterName,ParameterValue=prod-eks \
    ParameterKey=NodeInstanceType,ParameterValue=c5.xlarge \
  --capabilities CAPABILITY_IAM \
  --tags Key=Environment,Value=Production

# Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name prod-eks-cluster

# Check status
aws cloudformation describe-stacks \
  --stack-name prod-eks-cluster
```

### Update Stack (with Change Set)

```bash
# Create change set (preview changes)
aws cloudformation create-change-set \
  --stack-name prod-eks-cluster \
  --template-body file://eks-cluster-v2.yaml \
  --change-set-name update-node-count \
  --parameters \
    ParameterKey=NodeInstanceType,ParameterValue=c5.2xlarge

# Review change set
aws cloudformation describe-change-set \
  --stack-name prod-eks-cluster \
  --change-set-name update-node-count

# Execute change set
aws cloudformation execute-change-set \
  --stack-name prod-eks-cluster \
  --change-set-name update-node-count

# Wait for update
aws cloudformation wait stack-update-complete \
  --stack-name prod-eks-cluster
```

### Delete Stack

```bash
# Delete stack (deletes all resources)
aws cloudformation delete-stack \
  --stack-name prod-eks-cluster

# Wait for deletion
aws cloudformation wait stack-delete-complete \
  --stack-name prod-eks-cluster
```

---

## Nested Stacks (Modularity)

CloudFormation supports nested stacks for reusability:

### Parent Stack

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Parent stack for complete EKS infrastructure'

Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/templates/vpc-network.yaml
      Parameters:
        VpcCIDR: 10.0.0.0/16
        PublicSubnet1CIDR: 10.0.1.0/24
        PublicSubnet2CIDR: 10.0.2.0/24
      TimeoutInMinutes: 10
  
  EKSStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/templates/eks-cluster.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        SubnetIds: !GetAtt NetworkStack.Outputs.PrivateSubnetIds
      TimeoutInMinutes: 30

Outputs:
  ClusterName:
    Description: EKS Cluster Name
    Value: !GetAtt EKSStack.Outputs.ClusterName
```

### Child Stack (vpc-network.yaml)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'VPC and networking resources'

Parameters:
  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: MainVPC

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${AWS::StackName}-VpcId'
```

---

## StackSets (Multi-Account/Multi-Region)

Deploy same infrastructure across multiple accounts/regions:

```bash
# Create StackSet
aws cloudformation create-stack-set \
  --stack-set-name eks-baseline \
  --template-body file://eks-cluster.yaml \
  --parameters \
    ParameterKey=ClusterName,ParameterValue=eks \
  --capabilities CAPABILITY_IAM

# Deploy to multiple accounts
aws cloudformation create-stack-instances \
  --stack-set-name eks-baseline \
  --accounts 123456789012 234567890123 \
  --regions us-east-1 us-west-2 \
  --operation-preferences \
    FailureToleranceCount=0,MaxConcurrentCount=1

# Update StackSet
aws cloudformation update-stack-set \
  --stack-set-name eks-baseline \
  --template-body file://eks-cluster-v2.yaml

# Delete StackSet instances
aws cloudformation delete-stack-instances \
  --stack-set-name eks-baseline \
  --accounts 123456789012 \
  --regions us-east-1 \
  --no-retain-stacks
```

---

## Change Sets (Preview Before Update)

```bash
# Create change set
aws cloudformation create-change-set \
  --stack-name prod-eks \
  --template-body file://eks-updated.yaml \
  --change-set-name increase-nodes

# Describe changes
aws cloudformation describe-change-set \
  --stack-name prod-eks \
  --change-set-name increase-nodes

# Example output:
# Changes:
# - Action: Modify
#   ResourceType: AWS::EKS::Nodegroup
#   LogicalResourceId: NodeGroup
#   PhysicalResourceId: eks-node-group-abc123
#   Details:
#     - Target: ScalingConfig.DesiredSize
#       Before: 6
#       After: 12
#       Evaluation: Static
#       ChangeSource: DirectModification

# Execute if OK
aws cloudformation execute-change-set \
  --stack-name prod-eks \
  --change-set-name increase-nodes

# Or delete if not OK
aws cloudformation delete-change-set \
  --stack-name prod-eks \
  --change-set-name increase-nodes
```

---

## Common Interview Questions

### Q: "What is CloudFormation and when would you use it?"

**Answer:**
"CloudFormation is AWS's native Infrastructure as Code service. You define AWS resources in YAML or JSON templates, and CloudFormation provisions them. It's best when you're AWS-only and want day-0 support for new AWS services. CloudFormation has deep integration with AWS services like Service Catalog, Control Tower, and Organizations. I use it for AWS-specific features that Terraform doesn't support well, or when I need AWS-managed state (no S3 backend to manage). CloudFormation change sets let me preview changes before applying, which is great for production safety."

### Q: "What are the main differences between Terraform and CloudFormation?"

**Answer:**
"**Terraform:**
- Multi-cloud (AWS, Azure, GCP, 3000+ providers)
- HCL syntax (cleaner, more readable)
- Community modules (reusable components)
- You manage state (S3 + DynamoDB)
- Larger ecosystem and community

**CloudFormation:**
- AWS-only (but supports all AWS services immediately)
- JSON/YAML syntax (more verbose)
- Nested stacks for modularity
- AWS-managed state (nothing to manage)
- Native integration with AWS services

**When I choose Terraform:** Multi-cloud scenarios, when I want community modules, when I need better syntax (HCL vs YAML).

**When I choose CloudFormation:** AWS-only infrastructure, when I need day-0 support for new AWS services, when using AWS Control Tower or Service Catalog, when I want AWS to manage state."

### Q: "Explain CloudFormation change sets"

**Answer:**
"Change sets let you preview stack updates before applying them. When I update a CloudFormation template, I create a change set instead of updating directly. CloudFormation analyzes the changes and shows me exactly what will be modified, added, or deleted. For example, if I increase EKS node count from 6 to 12, the change set shows: 'Modify NodeGroup.ScalingConfig.DesiredSize from 6 to 12'. I review it, and if safe, execute the change set. If risky, I can delete the change set and revise the template. This is critical for production - never update blindly."

### Q: "What are CloudFormation StackSets?"

**Answer:**
"StackSets deploy the same CloudFormation template across multiple AWS accounts and regions from a single operation. For example, if I have 10 AWS accounts (dev, qa, prod, etc.) and want to deploy baseline security resources (IAM roles, GuardDuty, Config rules) to all, I create one StackSet. CloudFormation deploys to all accounts simultaneously. Updates propagate to all stack instances. This is perfect for multi-account governance, security baselines, or organization-wide standards."

---

## CloudFormation Stack Lifecycle

```
1. CREATE_IN_PROGRESS
   ↓ (resources being created)
2. CREATE_COMPLETE
   ↓ (stack created successfully)
3. UPDATE_IN_PROGRESS
   ↓ (resources being updated)
4. UPDATE_COMPLETE
   ↓ (stack updated successfully)
5. DELETE_IN_PROGRESS
   ↓ (resources being deleted)
6. DELETE_COMPLETE

Failure states:
- CREATE_FAILED → ROLLBACK_IN_PROGRESS → ROLLBACK_COMPLETE
- UPDATE_FAILED → UPDATE_ROLLBACK_IN_PROGRESS → UPDATE_ROLLBACK_COMPLETE
```

---

## Advanced CloudFormation Features

### Custom Resources (Lambda-backed)

```yaml
Resources:
  CustomResourceFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.9
      Handler: index.handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          import json
          import cfnresponse
          
          def handler(event, context):
              # Custom logic here
              response_data = {'Message': 'Success'}
              cfnresponse.send(event, context, cfnresponse.SUCCESS, response_data)
  
  CustomResource:
    Type: Custom::MyResource
    Properties:
      ServiceToken: !GetAtt CustomResourceFunction.Arn
      CustomProperty: value
```

### Drift Detection

```bash
# Detect drift (manual changes outside CloudFormation)
aws cloudformation detect-stack-drift \
  --stack-name prod-eks

# Check drift status
aws cloudformation describe-stack-resource-drifts \
  --stack-name prod-eks

# Example output:
# ResourceDriftStatus: MODIFIED
# PropertyDifferences:
#   - PropertyPath: /ScalingConfig/DesiredSize
#     Expected: 6
#     Actual: 8
#     DifferenceType: NOT_EQUAL
```

---

## Common Interview Questions

### Q: "How do you handle CloudFormation failures?"

**Answer:**
"CloudFormation automatically rolls back on failure. If resource creation fails (e.g., insufficient permissions, invalid configuration), CloudFormation deletes all resources created so far and returns the stack to the previous state. To troubleshoot: 1) Check stack events for the failed resource, 2) Review the specific error message, 3) Common issues are IAM permissions, resource limits, or dependencies. For updates, I always use change sets to preview changes before executing. If an update fails, CloudFormation rolls back to the previous stable state."

