# Phase 1: Hands-On Labs

## Practical Exercises for AWS Fundamentals

### Prerequisites
- AWS Account with free tier or budget
- AWS CLI installed and configured
- Estimated cost: $5-15 (mostly free tier eligible)
- Time: 3-4 hours

---

## Lab 1: IAM Policy & Cross-Account Access

### Exercise 1A: Create EC2 Instance Profile with S3 Read Access

**Objective:** EC2 instance can read from specific S3 bucket only

**Step 1: Create S3 Bucket**
```bash
BUCKET_NAME="devops-interview-lab-$(date +%s)"
aws s3 mb s3://$BUCKET_NAME --region us-east-1
echo "Bucket created: $BUCKET_NAME"

# Create test file
echo "Hello from S3" > test-file.txt
aws s3 cp test-file.txt s3://$BUCKET_NAME/
```

**Step 2: Create IAM Role**
```bash
# Create role with trust policy for EC2
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name EC2-S3-Reader \
  --assume-role-policy-document file://trust-policy.json
```

**Step 3: Create and Attach Policy**
```bash
# Create inline policy
cat > s3-read-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    },
    {
      "Sid": "S3ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    }
  ]
}
EOF

# Replace BUCKET_NAME
sed -i "s|BUCKET_NAME|$BUCKET_NAME|g" s3-read-policy.json

# Attach policy
aws iam put-role-policy \
  --role-name EC2-S3-Reader \
  --policy-name S3ReadPolicy \
  --policy-document file://s3-read-policy.json
```

**Step 4: Create Instance Profile**
```bash
# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-Reader

# Add role to profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-Reader \
  --role-name EC2-S3-Reader

# Wait for profile to be ready
sleep 10
```

**Step 5: Launch EC2 Instance**
```bash
# Get latest Amazon Linux 2 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

# Get default VPC and subnet
SUBNET_ID=$(aws ec2 describe-subnets \
  --query 'Subnets[0].SubnetId' \
  --output text)

# Get default security group
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=default" \
  --query 'SecurityGroups[0].GroupId' \
  --output text)

# Launch instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --iam-instance-profile Name=EC2-S3-Reader \
  --security-group-ids $SG_ID \
  --subnet-id $SUBNET_ID \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance launched: $INSTANCE_ID"

# Wait for instance to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is running"
```

**Step 6: Test S3 Access from Instance**
```bash
# Get instance IP
INSTANCE_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "Instance IP: $INSTANCE_IP"

# SSH into instance (may need key pair - simplified for EC2 Instance Connect)
# Using SSM Session Manager instead (simpler)

# Enable EC2 Instance metadata for testing
aws ssm start-session --target $INSTANCE_ID

# Inside session:
# 1. Test read access (should succeed)
aws s3 cp s3://$BUCKET_NAME/test-file.txt . --region us-east-1

# 2. Test write access (should fail)
echo "test" > test2.txt
aws s3 cp test2.txt s3://$BUCKET_NAME/ --region us-east-1  # Should fail!

# 3. Test access to other buckets (should fail)
aws s3 ls s3://some-other-bucket/  # Should fail!

# Exit session: exit
```

**Cleanup:**
```bash
# Terminate instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Remove IAM resources
aws iam delete-role-policy --role-name EC2-S3-Reader --policy-name S3ReadPolicy
aws iam remove-role-from-instance-profile \
  --instance-profile-name EC2-S3-Reader \
  --role-name EC2-S3-Reader
aws iam delete-instance-profile --instance-profile-name EC2-S3-Reader
aws iam delete-role --role-name EC2-S3-Reader

# Delete S3 bucket
aws s3 rm s3://$BUCKET_NAME/test-file.txt
aws s3 rb s3://$BUCKET_NAME

# Clean local files
rm -f trust-policy.json s3-read-policy.json test-file.txt
```

**Verification Checklist:**
- ✓ Instance can list bucket (ListBucket permission)
- ✓ Instance can read file (GetObject permission)
- ✓ Instance CANNOT write file (no PutObject permission)
- ✓ Instance CANNOT access other buckets (resource ARN constraint)
- ✓ Role has proper trust policy for EC2 service

---

### Exercise 1B: Cross-Account Role Assumption

**Setup: Two AWS Accounts Required**
- Account A (Production): 123456789012
- Account B (Development): 210987654321

**Account A Steps (Production):**

```bash
# Create role in Account A with trust relationship to Account B
cat > cross-account-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::210987654321:user/developer"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "UniqueExternalId12345"
        }
      }
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name CrossAccountEC2Reader \
  --assume-role-policy-document file://cross-account-trust.json

# Add EC2 read permissions
cat > cross-account-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name CrossAccountEC2Reader \
  --policy-name EC2ReadPolicy \
  --policy-document file://cross-account-policy.json

# Note the role ARN (will need for Account B)
ROLE_ARN=$(aws iam get-role \
  --role-name CrossAccountEC2Reader \
  --query 'Role.Arn' \
  --output text)

echo "Role ARN: $ROLE_ARN"
```

**Account B Steps (Development):**

```bash
# Create policy allowing assumption of role in Account A
cat > assume-role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/CrossAccountEC2Reader",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "UniqueExternalId12345"
        }
      }
    }
  ]
}
EOF

# Attach policy to developer user in Account B
aws iam put-user-policy \
  --user-name developer \
  --policy-name AssumeProductionRole \
  --policy-document file://assume-role-policy.json

# Developer assumes role
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/CrossAccountEC2Reader \
  --role-session-name development-session \
  --external-id UniqueExternalId12345 \
  --duration-seconds 3600

# Output will include:
# - AccessKeyId
# - SecretAccessKey
# - SessionToken
# - Expiration

# Set these in environment
export AWS_ACCESS_KEY_ID=<temporary-key>
export AWS_SECRET_ACCESS_KEY=<temporary-secret>
export AWS_SESSION_TOKEN=<session-token>

# Now can access Account A resources
aws ec2 describe-instances --region us-east-1

# Verify in CloudTrail of Account A shows the cross-account access
```

**Verification Checklist:**
- ✓ External ID prevents confused deputy problem
- ✓ Developer in Account B can assume role in Account A
- ✓ Temporary credentials have expiration (3600 seconds default)
- ✓ CloudTrail in Account A logs the AssumeRole action
- ✓ Developer can access EC2 instances in Account A
- ✓ Developer CANNOT perform other actions (only EC2 read)

---

## Lab 2: EC2, Auto Scaling & Load Balancing

### Exercise 2A: Launch Template & Auto Scaling Group

**Objective:** Create ASG with ALB that auto-scales based on CPU

**Step 1: Create Launch Template**
```bash
cat > launch-template.json << 'EOF'
{
  "ImageId": "ami-0c55b159cbfafe1f0",
  "InstanceType": "t2.micro",
  "KeyName": "my-key-pair",
  "Monitoring": {
    "Enabled": true
  },
  "UserData": "IyEvYmluL2Jhc2gKYXB0IHVwZGF0ZQphcHQgaW5zdGFsbCAteSBuZ2lueApzeXN0ZW1jdGwgc3RhcnQgbmdpbng="
}
EOF

# UserData is base64 encoded:
# #!/bin/bash
# apt update
# apt install -y nginx
# systemctl start nginx

aws ec2 create-launch-template \
  --launch-template-name devops-lab-template \
  --launch-template-data file://launch-template.json
```

**Step 2: Create Target Group**
```bash
# Get default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name devops-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --health-check-enabled \
  --health-check-path / \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

echo "Target Group: $TG_ARN"
```

**Step 3: Create Application Load Balancer**
```bash
# Get default subnets
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].SubnetId' \
  --output text)

# Create security group for ALB
SG_ID=$(aws ec2 create-security-group \
  --group-name devops-alb-sg \
  --description "ALB security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name devops-alb \
  --subnets $SUBNET_IDS \
  --security-groups $SG_ID \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

echo "ALB ARN: $ALB_ARN"

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

echo "ALB DNS: $ALB_DNS"
```

**Step 4: Create Listener**
```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

**Step 5: Create Auto Scaling Group**
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name devops-asg \
  --launch-template "LaunchTemplateName=devops-lab-template,Version=\$Latest" \
  --min-size 2 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$SUBNET_IDS" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB \
  --health-check-grace-period 300
```

**Step 6: Create Scaling Policy**
```bash
# Target tracking policy for CPU
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name devops-asg \
  --policy-name cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 50.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 600
  }'
```

**Step 7: Verify Setup**
```bash
# Check ASG status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names devops-asg \
  --query 'AutoScalingGroups[0].[Instances,DesiredCapacity,MinSize,MaxSize]'

# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN

# Access ALB (wait for health check passes)
sleep 30
curl http://$ALB_DNS
```

**Stress Test (Optional):**
```bash
# Install Apache Bench on your local machine
# ab -n 10000 -c 100 http://$ALB_DNS/

# Monitor CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average \
  --dimensions Name=AutoScalingGroupName,Value=devops-asg
```

**Cleanup:**
```bash
# Delete ASG (instances will terminate)
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name devops-asg \
  --force-delete

# Delete ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

# Delete target group
aws elbv2 delete-target-group --target-group-arn $TG_ARN

# Delete security group (wait for ALB deletion)
sleep 10
aws ec2 delete-security-group --group-id $SG_ID

# Delete launch template
aws ec2 delete-launch-template --launch-template-name devops-lab-template
```

---

## Lab 3: S3 Bucket with Lifecycle & Encryption

### Exercise 3A: S3 Lifecycle Policy Implementation

**Objective:** Create bucket with versioning, encryption, and lifecycle policy

**Step 1: Create S3 Bucket**
```bash
BUCKET_NAME="devops-lifecycle-lab-$(date +%s)"

aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region us-east-1

echo "Bucket created: $BUCKET_NAME"
```

**Step 2: Enable Versioning**
```bash
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# Verify versioning
aws s3api get-bucket-versioning --bucket $BUCKET_NAME
```

**Step 3: Enable Default Encryption**
```bash
cat > encryption-config.json << 'EOF'
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }
  ]
}
EOF

aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration file://encryption-config.json

# Verify encryption
aws s3api get-bucket-encryption --bucket $BUCKET_NAME
```

**Step 4: Create Lifecycle Policy**
```bash
cat > lifecycle-policy.json << 'EOF'
{
  "Rules": [
    {
      "Id": "Archive old logs",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    },
    {
      "Id": "Delete incomplete multipart uploads",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET_NAME \
  --lifecycle-configuration file://lifecycle-policy.json

# Verify lifecycle
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET_NAME
```

**Step 5: Block Public Access**
```bash
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Verify
aws s3api get-public-access-block --bucket $BUCKET_NAME
```

**Step 6: Upload Test Data**
```bash
# Create test files
mkdir -p test-data/logs
for i in {1..5}; do
  echo "Log file $i" > test-data/logs/app-log-$i.txt
done

# Upload to logs/ prefix (will be managed by lifecycle)
aws s3 sync test-data/logs s3://$BUCKET_NAME/logs/

# Verify uploads
aws s3 ls s3://$BUCKET_NAME/logs/ --recursive

# Check versioning
aws s3api list-object-versions --bucket $BUCKET_NAME \
  --prefix "logs/" \
  --query 'Versions[] | sort_by(@, &LastModified) | reverse(@)' \
  --output table
```

**Step 7: Test Encryption Verification**
```bash
# Upload file and check encryption metadata
aws s3 cp test-data/logs/app-log-1.txt s3://$BUCKET_NAME/logs/test.txt

# Get object metadata to confirm encryption
aws s3api head-object --bucket $BUCKET_NAME --key logs/test.txt \
  --query '[ServerSideEncryption, StorageClass]'
```

**Cleanup:**
```bash
# Delete all versions and objects
aws s3 rm s3://$BUCKET_NAME --recursive

# Delete bucket
aws s3api delete-bucket --bucket $BUCKET_NAME

# Remove test data
rm -rf test-data encryption-config.json lifecycle-policy.json
```

---

## Lab 4: RDS Multi-AZ with Automated Backups

### Exercise 4A: RDS Setup with High Availability

**Objective:** Create Multi-AZ RDS instance with automated backups and read replica

**Step 1: Create Security Group for RDS**
```bash
# Get default VPC
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

# Create security group
RDS_SG=$(aws ec2 create-security-group \
  --group-name devops-rds-sg \
  --description "RDS security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

# Allow MySQL from anywhere (not recommended for production!)
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp \
  --port 3306 \
  --cidr 0.0.0.0/0

echo "Security group: $RDS_SG"
```

**Step 2: Create RDS Subnet Group**
```bash
# Get subnets
SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone]' \
  --output text)

# Extract subnet IDs from different AZs
SUBNET_IDS=$(echo "$SUBNETS" | awk '{print $1}' | head -2 | tr '\n' ' ')

# Create DB subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name devops-rds-subnet \
  --db-subnet-group-description "RDS subnet group" \
  --subnet-ids $SUBNET_IDS
```

**Step 3: Create Multi-AZ RDS Instance**
```bash
aws rds create-db-instance \
  --db-instance-identifier devops-mysql-primary \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password MyPassword123 \
  --allocated-storage 20 \
  --db-subnet-group-name devops-rds-subnet \
  --vpc-security-group-ids $RDS_SG \
  --multi-az \
  --backup-retention-period 7 \
  --backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --enable-cloudwatch-logs-exports error,general,slowquery \
  --enable-iam-database-authentication \
  --storage-encrypted \
  --kms-key-id alias/aws/rds

echo "RDS instance creating... (takes ~5-10 minutes)"

# Wait for instance to be ready
aws rds wait db-instance-available \
  --db-instance-identifier devops-mysql-primary

echo "RDS instance is available"
```

**Step 4: Get Connection Details**
```bash
# Get endpoint
RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier devops-mysql-primary \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)

echo "RDS Endpoint: $RDS_ENDPOINT"

# Get instance details
aws rds describe-db-instances \
  --db-instance-identifier devops-mysql-primary \
  --query 'DBInstances[0].[DBInstanceStatus,MultiAZ,BackupRetentionPeriod,StorageEncrypted]'
```

**Step 5: Create Cross-Region Read Replica**
```bash
# Create read replica in different region
aws rds create-db-instance-read-replica \
  --db-instance-identifier devops-mysql-replica-west \
  --source-db-instance-identifier devops-mysql-primary \
  --availability-zone us-west-2a \
  --publicly-accessible false \
  --region us-west-2

echo "Read replica creating in us-west-2..."

# Wait in different region
aws rds wait db-instance-available \
  --db-instance-identifier devops-mysql-replica-west \
  --region us-west-2

echo "Read replica is available"
```

**Step 6: Test Replication**
```bash
# List all instances and replicas
aws rds describe-db-instances \
  --query 'DBInstances[] | sort_by(@, &DBInstanceIdentifier)' \
  --output table

# Check replication lag
aws rds describe-db-instances \
  --db-instance-identifier devops-mysql-replica-west \
  --region us-west-2 \
  --query 'DBInstances[0].ReplicationLag' \
  --output text
```

**Step 7: Test Backup & Restore**
```bash
# Create manual snapshot
SNAPSHOT_ID="devops-mysql-snapshot-$(date +%s)"

aws rds create-db-snapshot \
  --db-instance-identifier devops-mysql-primary \
  --db-snapshot-identifier $SNAPSHOT_ID

echo "Snapshot creating: $SNAPSHOT_ID"

# Wait for snapshot
aws rds wait db-snapshot-available \
  --db-snapshot-identifier $SNAPSHOT_ID

# List snapshots
aws rds describe-db-snapshots \
  --query 'DBSnapshots[] | sort_by(@, &CreateTime)' \
  --output table

# Restore from snapshot (don't restore, just show command)
# aws rds restore-db-instance-from-db-snapshot \
#   --db-instance-identifier devops-mysql-restored \
#   --db-snapshot-identifier $SNAPSHOT_ID
```

**Cleanup:**
```bash
# Delete read replica
aws rds delete-db-instance \
  --db-instance-identifier devops-mysql-replica-west \
  --skip-final-snapshot \
  --region us-west-2

# Wait for deletion
aws rds wait db-instance-deleted \
  --db-instance-identifier devops-mysql-replica-west \
  --region us-west-2

# Delete primary (keep snapshot)
aws rds delete-db-instance \
  --db-instance-identifier devops-mysql-primary \
  --skip-final-snapshot

# Wait for deletion
aws rds wait db-instance-deleted \
  --db-instance-identifier devops-mysql-primary

# Delete DB subnet group
aws rds delete-db-subnet-group \
  --db-subnet-group-name devops-rds-subnet

# Delete security group
sleep 10
aws ec2 delete-security-group --group-id $RDS_SG

# Delete snapshot
aws rds delete-db-snapshot --db-snapshot-identifier $SNAPSHOT_ID
```

---

## Lab 5: VPC, NAT Gateway & Route53

### Exercise 5A: Multi-Tier VPC Architecture

**Objective:** Create VPC with public/private subnets, NAT Gateway, and Route53

**Step 1: Create VPC**
```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' \
  --output text)

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames

echo "VPC created: $VPC_ID"
```

**Step 2: Create Subnets**
```bash
# Public Subnet in AZ1
PUBLIC_SUBNET_AZ1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' \
  --output text)

# Private Subnet in AZ1
PRIVATE_SUBNET_AZ1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' \
  --output text)

# Public Subnet in AZ2
PUBLIC_SUBNET_AZ2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1b \
  --query 'Subnet.SubnetId' \
  --output text)

# Private Subnet in AZ2
PRIVATE_SUBNET_AZ2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.4.0/24 \
  --availability-zone us-east-1b \
  --query 'Subnet.SubnetId' \
  --output text)

echo "Public Subnets: $PUBLIC_SUBNET_AZ1, $PUBLIC_SUBNET_AZ2"
echo "Private Subnets: $PRIVATE_SUBNET_AZ1, $PRIVATE_SUBNET_AZ2"
```

**Step 3: Create Internet Gateway**
```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

echo "Internet Gateway: $IGW_ID"
```

**Step 4: Create and Configure Route Tables**
```bash
# Public route table
PUBLIC_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Route all traffic to IGW
aws ec2 create-route \
  --route-table-id $PUBLIC_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associate public subnets
aws ec2 associate-route-table \
  --subnet-id $PUBLIC_SUBNET_AZ1 \
  --route-table-id $PUBLIC_RT

aws ec2 associate-route-table \
  --subnet-id $PUBLIC_SUBNET_AZ2 \
  --route-table-id $PUBLIC_RT

# Private route table
PRIVATE_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Associate private subnets
aws ec2 associate-route-table \
  --subnet-id $PRIVATE_SUBNET_AZ1 \
  --route-table-id $PRIVATE_RT

aws ec2 associate-route-table \
  --subnet-id $PRIVATE_SUBNET_AZ2 \
  --route-table-id $PRIVATE_RT

echo "Route Tables: Public=$PUBLIC_RT, Private=$PRIVATE_RT"
```

**Step 5: Create NAT Gateway**
```bash
# Allocate Elastic IP for NAT Gateway
EIP=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'PublicIp' \
  --output text)

# Create NAT Gateway in public subnet
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id $PUBLIC_SUBNET_AZ1 \
  --allocation-id $(aws ec2 describe-addresses \
    --public-ips $EIP \
    --query 'Addresses[0].AllocationId' \
    --output text) \
  --query 'NatGateway.NatGatewayId' \
  --output text)

# Wait for NAT Gateway
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

echo "NAT Gateway: $NAT_GW with EIP: $EIP"
```

**Step 6: Configure Private Route Table for NAT**
```bash
# Add route to NAT Gateway in private route table
aws ec2 create-route \
  --route-table-id $PRIVATE_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW

# Verify routes
aws ec2 describe-route-tables \
  --route-table-ids $PUBLIC_RT $PRIVATE_RT \
  --output table
```

**Step 7: Create VPC Flow Logs**
```bash
# Create IAM role for VPC Flow Logs
cat > flow-logs-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

FLOW_ROLE=$(aws iam create-role \
  --role-name vpc-flow-logs-role \
  --assume-role-policy-document file://flow-logs-trust.json \
  --query 'Role.Arn' \
  --output text)

# Create CloudWatch log group
LOG_GROUP="/aws/vpc/flowlogs/devops-lab"
aws logs create-log-group --log-group-name $LOG_GROUP

# Attach policy
cat > flow-logs-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name vpc-flow-logs-role \
  --policy-name vpc-flow-logs-policy \
  --policy-document file://flow-logs-policy.json

# Enable VPC Flow Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name $LOG_GROUP \
  --deliver-logs-permission-role-name vpc-flow-logs-role
```

**Step 8: Test VPC Connectivity**
```bash
# Launch EC2 instance in public subnet
PUBLIC_SG=$(aws ec2 create-security-group \
  --group-name public-sg \
  --description "Public security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $PUBLIC_SG \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# Get AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

# Launch instance
PUBLIC_INSTANCE=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --subnet-id $PUBLIC_SUBNET_AZ1 \
  --security-group-ids $PUBLIC_SG \
  --query 'Instances[0].InstanceId' \
  --output text)

# Wait for running
aws ec2 wait instance-running --instance-ids $PUBLIC_INSTANCE

echo "Public instance: $PUBLIC_INSTANCE"
```

**Cleanup:**
```bash
# Terminate instance
aws ec2 terminate-instances --instance-ids $PUBLIC_INSTANCE
aws ec2 wait instance-terminated --instance-ids $PUBLIC_INSTANCE

# Delete security group
sleep 10
aws ec2 delete-security-group --group-id $PUBLIC_SG

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW

# Wait for NAT deletion
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_GW

# Release Elastic IP
aws ec2 release-address --public-ip $EIP

# Delete route tables (detach subnets first)
aws ec2 disassociate-route-table --association-id $(aws ec2 describe-route-tables \
  --route-table-ids $PUBLIC_RT --query 'RouteTables[0].Associations[0].RouteTableAssociationId' --output text)

aws ec2 delete-route-table --route-table-id $PUBLIC_RT
aws ec2 delete-route-table --route-table-id $PRIVATE_RT

# Delete subnets
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_AZ1
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET_AZ2
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_AZ1
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_AZ2

# Detach and delete IGW
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# Delete VPC Flow Logs
aws logs delete-log-group --log-group-name $LOG_GROUP
aws iam delete-role-policy --role-name vpc-flow-logs-role --policy-name vpc-flow-logs-policy
aws iam delete-role --role-name vpc-flow-logs-role

# Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID

# Clean local files
rm -f flow-logs-trust.json flow-logs-policy.json
```

---

## Lab Verification Checklist

### After Completing All Labs

- [ ] IAM Exercise 1A: EC2 can read S3, cannot write or access other buckets
- [ ] IAM Exercise 1B: Developer assumes role across accounts with external ID
- [ ] EC2 Exercise 2A: ALB distributes traffic across 2+ instances, scales based on CPU
- [ ] S3 Exercise 3A: Objects transition through storage classes based on lifecycle policy
- [ ] RDS Exercise 4A: Multi-AZ primary with cross-region read replica, automated backups
- [ ] VPC Exercise 5A: Private instances route through NAT Gateway to internet
- [ ] All CloudTrail logs show actions performed
- [ ] All resources cleaned up successfully (no orphaned resources)

---

## Estimated Costs

| Service | Est. Cost | Notes |
|---------|-----------|-------|
| EC2 (2 x t2.micro, 2 hrs) | $0.02 | Free tier eligible |
| RDS (t3.micro, 2 hrs) | $0.10 | Free tier eligible for some accounts |
| ALB (2 hrs) | $0.30 | ~$0.15/hr |
| NAT Gateway (2 hrs) | $0.03 | $0.015/hour |
| Data transfer | $0.50 | Out-of-region transfer |
| **Total** | **~$0.95** | Mostly free tier |

**Note:** Enable AWS Cost Anomaly Detection to catch unexpected charges

---

**Total Lab Time: 3-4 hours**
