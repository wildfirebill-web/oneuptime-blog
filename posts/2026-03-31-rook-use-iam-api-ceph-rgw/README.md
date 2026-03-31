# How to Use the IAM API with Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, IAM, Authentication, S3

Description: Learn how to use the IAM API with Ceph RGW for managing users, roles, policies, and temporary credentials compatible with AWS IAM workflows.

---

## Overview

Ceph RGW implements a subset of the AWS Identity and Access Management (IAM) API, enabling you to manage roles, policies, and OIDC identity providers. This allows applications designed for AWS IAM to work with Ceph RGW using the same API calls, including assuming roles and generating temporary credentials via STS.

## Enabling the IAM API

The IAM API is enabled by default in recent Ceph versions. Configure the IAM endpoint in ceph.conf if needed:

```bash
# Verify IAM API is accessible
curl http://rgw-host:80/?Action=ListUsers \
  -H "Authorization: AWS4-HMAC-SHA256 ..."
```

## Creating an IAM Role

IAM roles in Ceph RGW allow cross-account and application-level access:

```bash
# Create trust policy for the role
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam:::user/app-user"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role using AWS CLI against RGW
aws --endpoint-url http://rgw-host:80 iam create-role \
  --role-name s3-read-role \
  --assume-role-policy-document file://trust-policy.json
```

## Attaching a Policy to a Role

```bash
# Create a permissions policy
cat > s3-read-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::mybucket",
        "arn:aws:s3:::mybucket/*"
      ]
    }
  ]
}
EOF

# Attach the policy to the role
aws --endpoint-url http://rgw-host:80 iam put-role-policy \
  --role-name s3-read-role \
  --policy-name S3ReadAccess \
  --policy-document file://s3-read-policy.json
```

## Assuming a Role via STS

```python
import boto3

# Create STS client
sts = boto3.client(
    'sts',
    endpoint_url='http://rgw-host:80',
    aws_access_key_id='MYACCESSKEY',
    aws_secret_access_key='MYSECRETKEY',
    region_name='us-east-1'
)

# Assume the role
response = sts.assume_role(
    RoleArn='arn:aws:iam:::role/s3-read-role',
    RoleSessionName='my-session',
    DurationSeconds=3600
)

# Use the temporary credentials
creds = response['Credentials']
s3 = boto3.client(
    's3',
    endpoint_url='http://rgw-host:80',
    aws_access_key_id=creds['AccessKeyId'],
    aws_secret_access_key=creds['SecretAccessKey'],
    aws_session_token=creds['SessionToken']
)

# Now use s3 with the role's permissions
s3.list_objects_v2(Bucket='mybucket')
```

## Configuring OIDC Provider

Allow tokens from an OIDC provider (e.g., Kubernetes service accounts) to assume roles:

```bash
# Create an OIDC provider
aws --endpoint-url http://rgw-host:80 iam create-open-id-connect-provider \
  --url https://kubernetes.default.svc \
  --thumbprint-list "abc123..." \
  --client-id-list sts.amazonaws.com
```

## Listing IAM Resources

```bash
# List roles
aws --endpoint-url http://rgw-host:80 iam list-roles

# List role policies
aws --endpoint-url http://rgw-host:80 iam list-role-policies \
  --role-name s3-read-role
```

## Summary

Ceph RGW's IAM API implementation supports role creation, policy attachment, and STS token generation compatible with AWS IAM clients. Use IAM roles to enable fine-grained access control and temporary credential issuance. The OIDC provider support allows Kubernetes workloads using pod identity to authenticate against Ceph RGW using the same patterns as AWS EKS with IAM Roles for Service Accounts.
