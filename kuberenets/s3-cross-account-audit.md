# Cross-Account S3 IAM Configuration Audit

## Current Configuration

### QA Account (Bucket Owner: 332728166114)
**Bucket Policy:**
- Allows: Prod account role (arn:aws:iam::344760941228:role/CrossAccountS3Role)
- Actions: s3:GetObject, s3:ListBucket
- Resources: arn:aws:s3:::preload-sample-data and arn:aws:s3:::preload-sample-data/*
- Status: ✅ CORRECT

### PROD Account (Accessor: 344760941228)
**Role: CrossAccountS3Role**
- Trust Policy: Only allows EC2 service to assume the role
- Inline Policy: AllowCrossAccountS3
  - Actions: s3:GetObject, s3:ListBucket
  - Resources: arn:aws:s3:::preload-sample-data and arn:aws:s3:::preload-sample-data/*
  - Status: ✅ CORRECT

---

## Issues Found

### Issue 1: ⚠️ Trust Policy Too Restrictive
**Problem:** The CrossAccountS3Role only allows EC2 to assume it
```json
{
  "Effect": "Allow",
  "Principal": { "Service": "ec2.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

**Why This Matters:** 
- If you're using IAM Users/Roles directly (not EC2), you CANNOT assume this role
- You can only use this role from EC2 instances

**Solution:** Add your IAM user/role to the trust policy

---

### Issue 2: ⚠️ Missing IAM User/Role Permissions
**Problem:** Your IAM user in PROD account cannot assume the CrossAccountS3Role

**Solution:** Add a policy to your IAM user allowing them to assume the role

---

## What Works Currently
✅ EC2 instances in PROD account CAN access the S3 bucket in QA
✅ Bucket policy correctly trusts the PROD role
✅ S3 permissions on the PROD role are correct

## What Doesn't Work Currently
❌ IAM users in PROD account trying to access bucket directly
❌ IAM users in PROD account assuming the CrossAccountS3Role

