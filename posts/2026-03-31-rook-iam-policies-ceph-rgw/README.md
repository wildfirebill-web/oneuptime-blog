# How to Configure IAM Policies for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, IAM, S3

Description: Learn how to create and apply AWS-compatible IAM policies in Ceph RGW to control fine-grained access to S3 buckets and objects.

---

## Overview

Ceph RGW implements AWS-compatible IAM policies that let you define granular permissions for users and roles. Policies can be attached to users (inline or managed), applied as bucket policies, or used as permission boundaries. This guide walks through creating and managing IAM policies in Ceph RGW using standard AWS tooling.

## Step 1 - Enable IAM Support in RGW

```bash
# Create an IAM admin user with full IAM capabilities
radosgw-admin user create \
  --uid=iam-admin \
  --display-name="IAM Admin" \
  --caps="users=*;buckets=*;roles=*;policies=*;oidc-provider=*"

# Export credentials
IAM_ACCESS=$(radosgw-admin user info --uid=iam-admin | jq -r '.keys[0].access_key')
IAM_SECRET=$(radosgw-admin user info --uid=iam-admin | jq -r '.keys[0].secret_key')

# Configure AWS CLI profile for RGW IAM operations
aws configure --profile rgw-iam
# Access Key: $IAM_ACCESS
# Secret Key: $IAM_SECRET
# Region: us-east-1
# Output: json
```

## Step 2 - Create a Managed IAM Policy

```bash
# Create an S3 read-only policy
cat > s3-readonly-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:ListBucketVersions",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::app-data-bucket",
        "arn:aws:s3:::app-data-bucket/*"
      ]
    }
  ]
}
EOF

# Create the managed policy in RGW
aws --endpoint-url http://rgw.example.com:7480 \
  --profile rgw-iam \
  iam create-policy \
  --policy-name S3ReadOnly \
  --policy-document file://s3-readonly-policy.json
```

## Step 3 - Attach Policies to Users and Roles

```bash
# Create an application user
radosgw-admin user create --uid=app-user --display-name="App User"

# Attach managed policy to user
aws --endpoint-url http://rgw.example.com:7480 \
  --profile rgw-iam \
  iam attach-user-policy \
  --user-name app-user \
  --policy-arn "arn:aws:iam:::policy/S3ReadOnly"

# Attach inline policy to a role
aws --endpoint-url http://rgw.example.com:7480 \
  --profile rgw-iam \
  iam put-user-policy \
  --user-name app-user \
  --policy-name AllowListBuckets \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:ListAllMyBuckets","Resource":"*"}]}'
```

## Step 4 - Create Bucket Policies

Bucket policies apply permissions directly to a specific bucket:

```bash
# Create a bucket policy allowing a specific role to read objects
cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRoleRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam:::role/analytics-role"
      },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::analytics-data",
        "arn:aws:s3:::analytics-data/*"
      ]
    },
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::analytics-data/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalType": ["AssumedRole", "IAMUser"]
        }
      }
    }
  ]
}
EOF

# Apply the bucket policy
aws --endpoint-url http://rgw.example.com:7480 \
  --profile rgw-iam \
  s3api put-bucket-policy \
  --bucket analytics-data \
  --policy file://bucket-policy.json
```

## Step 5 - Use Condition Keys in Policies

```bash
# Restrict access to specific IP ranges
cat > ip-restricted-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::secure-bucket", "arn:aws:s3:::secure-bucket/*"],
      "Condition": {
        "IpAddress": {"aws:SourceIp": ["10.0.0.0/8", "192.168.1.0/24"]},
        "Bool": {"aws:SecureTransport": "true"}
      }
    }
  ]
}
EOF

aws --endpoint-url http://rgw.example.com:7480 \
  --profile rgw-iam \
  s3api put-bucket-policy \
  --bucket secure-bucket \
  --policy file://ip-restricted-policy.json
```

## Step 6 - Verify and Audit Policies

```bash
# List all managed policies
aws --endpoint-url http://rgw.example.com:7480 \
  --profile rgw-iam \
  iam list-policies --scope Local

# Simulate a policy evaluation
aws --endpoint-url http://rgw.example.com:7480 \
  --profile rgw-iam \
  iam simulate-principal-policy \
  --policy-source-arn "arn:aws:iam:::user/app-user" \
  --action-names "s3:GetObject" \
  --resource-arns "arn:aws:s3:::app-data-bucket/myfile.txt"
```

## Summary

Ceph RGW's IAM policy system is fully compatible with AWS IAM, supporting managed policies, inline policies, and bucket policies. Using condition keys such as `aws:SourceIp` and `aws:SecureTransport` enables fine-grained access controls. The `simulate-principal-policy` call helps validate policy logic before deployment, reducing the risk of misconfiguration.
