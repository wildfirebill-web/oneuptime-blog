# How to Configure STS (Security Token Service) for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, STS, Security

Description: Learn how to enable and configure the AWS-compatible Security Token Service in Ceph RGW for issuing temporary credentials via AssumeRole and AssumeRoleWithWebIdentity.

---

## Overview

Ceph RGW includes an AWS-compatible Security Token Service (STS) that issues short-lived credentials. This is the foundation for role-based S3 access, OIDC federation, and applications that need temporary credentials without long-lived access keys. STS in Ceph RGW supports `AssumeRole`, `AssumeRoleWithWebIdentity`, and `GetSessionToken`.

## Step 1 - Enable STS in RGW Configuration

```bash
# Enable STS authentication support
ceph config set client.rgw.my-store rgw_s3_auth_use_sts true

# Set the STS encryption key (must be at least 16 bytes)
ceph config set client.rgw.my-store rgw_sts_key "my-sts-encryption-key-32bytes!!"

# Set the token expiry window
ceph config set client.rgw.my-store rgw_sts_max_session_duration 43200

# Verify STS is enabled
ceph config get client.rgw.my-store rgw_s3_auth_use_sts
```

For Rook-managed RGW:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- \
  ceph config set client.rgw.my-store rgw_s3_auth_use_sts true
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- \
  ceph config set client.rgw.my-store rgw_sts_key "my-sts-encryption-key-32bytes!!"
```

## Step 2 - Create an IAM User for STS Operations

The AssumeRole caller needs an RGW user with the `roles` capability:

```bash
# Create an IAM user
radosgw-admin user create \
  --uid=iam-admin \
  --display-name="IAM Admin" \
  --caps="roles=*;oidc-provider=*"

# Get the access key and secret
radosgw-admin user info --uid=iam-admin | jq '.keys'
```

## Step 3 - Create IAM Roles

```bash
# Create a trust policy for role assumption
cat > assume-role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam:::user/iam-admin"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role in RGW
AWS_ACCESS_KEY_ID=<IAM_ADMIN_ACCESS_KEY> \
AWS_SECRET_ACCESS_KEY=<IAM_ADMIN_SECRET_KEY> \
aws --endpoint-url http://rgw.example.com:7480 \
  iam create-role \
  --role-name app-role \
  --assume-role-policy-document file://assume-role-policy.json

# Attach a permissions policy
aws --endpoint-url http://rgw.example.com:7480 \
  iam put-role-policy \
  --role-name app-role \
  --policy-name s3-fullaccess \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"s3:*","Resource":"*"}]}'
```

## Step 4 - Use AssumeRole to Get Temporary Credentials

```python
import boto3

# Configure the STS client with the IAM admin credentials
sts_client = boto3.client(
    "sts",
    endpoint_url="http://rgw.example.com:7480",
    aws_access_key_id="<IAM_ADMIN_ACCESS_KEY>",
    aws_secret_access_key="<IAM_ADMIN_SECRET_KEY>",
    region_name="us-east-1"
)

# Assume the role
response = sts_client.assume_role(
    RoleArn="arn:aws:iam:::role/app-role",
    RoleSessionName="my-app-session",
    DurationSeconds=3600
)

# Use the temporary credentials
temp_creds = response["Credentials"]
s3_client = boto3.client(
    "s3",
    endpoint_url="http://rgw.example.com:7480",
    aws_access_key_id=temp_creds["AccessKeyId"],
    aws_secret_access_key=temp_creds["SecretAccessKey"],
    aws_session_token=temp_creds["SessionToken"]
)
buckets = s3_client.list_buckets()
print(buckets)
```

## Step 5 - Use GetSessionToken for MFA-like Flows

```bash
# Get a session token using existing credentials
aws --endpoint-url http://rgw.example.com:7480 \
  sts get-session-token \
  --duration-seconds 3600

# Returns temporary credentials valid for 1 hour
```

## Step 6 - Monitor STS Usage

```bash
# Check RGW logs for STS operations
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=200 \
  | grep -i "sts\|AssumeRole\|session_token"

# Monitor via Prometheus metrics
curl http://rgw.example.com:9283/metrics | grep rgw_sts
```

## Summary

Ceph RGW's STS implementation provides AWS-compatible temporary credential issuance via `AssumeRole` and `AssumeRoleWithWebIdentity`. Enabling STS requires setting the encryption key and creating IAM users with the appropriate capabilities. Applications use temporary credentials for short-lived S3 access, reducing the risk associated with long-lived access keys stored in application configurations.
