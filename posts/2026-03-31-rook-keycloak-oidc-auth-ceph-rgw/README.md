# How to Configure Keycloak OIDC Authentication for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Keycloak, OIDC

Description: Learn how to configure Ceph RGW to use Keycloak as an OIDC provider for S3 access via STS AssumeRoleWithWebIdentity for federated authentication.

---

## Overview

Keycloak integration with Ceph RGW uses the OpenID Connect (OIDC) protocol and the AWS STS `AssumeRoleWithWebIdentity` API. Applications obtain JWT tokens from Keycloak and exchange them for temporary S3 credentials via RGW's STS endpoint. This avoids distributing long-lived access keys and enables fine-grained role-based access.

## Step 1 - Configure Keycloak

Create a client in Keycloak for Ceph RGW:

```bash
# Using Keycloak Admin CLI
kcadm.sh config credentials \
  --server https://keycloak.example.com/auth \
  --realm master --user admin --password admin

# Create a client for RGW
kcadm.sh create clients -r myrealm -s clientId=ceph-rgw \
  -s enabled=true \
  -s protocol=openid-connect \
  -s publicClient=false \
  -s standardFlowEnabled=false \
  -s serviceAccountsEnabled=true \
  -s "attributes.\"access.token.signed.response.alg\"=RS256"

# Create roles that map to RGW permissions
kcadm.sh create roles -r myrealm -s name=storage-read
kcadm.sh create roles -r myrealm -s name=storage-write
```

## Step 2 - Enable STS in Ceph RGW

```bash
# Enable the STS module in RGW
ceph config set client.rgw.my-store rgw_s3_auth_use_sts true
ceph config set client.rgw.my-store rgw_sts_key "abcdefghij1234567890abcdefghij12"
ceph config set client.rgw.my-store rgw_max_chunk_size 524288

# For Rook:
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- \
  ceph config set client.rgw.my-store rgw_s3_auth_use_sts true
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- \
  ceph config set client.rgw.my-store rgw_sts_key "abcdefghij1234567890abcdefghij12"
```

## Step 3 - Create an OIDC Provider in RGW

Register the Keycloak OIDC provider with RGW using the AWS CLI:

```bash
# Get the Keycloak OIDC thumbprint
THUMBPRINT=$(openssl s_client -connect keycloak.example.com:443 \
  -showcerts </dev/null 2>/dev/null | \
  openssl x509 -fingerprint -noout -sha1 | \
  sed 's/://g' | awk -F= '{print $2}' | tr '[:upper:]' '[:lower:]')

# Create the OIDC provider in RGW
aws --endpoint-url http://rgw.example.com:7480 \
  iam create-open-id-connect-provider \
  --url "https://keycloak.example.com/auth/realms/myrealm" \
  --client-id-list "ceph-rgw" \
  --thumbprint-list "${THUMBPRINT}"
```

## Step 4 - Create IAM Roles for RGW

```bash
# Create a trust policy document
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam:::oidc-provider/keycloak.example.com/auth/realms/myrealm"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "keycloak.example.com/auth/realms/myrealm:aud": "ceph-rgw"
        }
      }
    }
  ]
}
EOF

# Create the role in RGW
aws --endpoint-url http://rgw.example.com:7480 \
  iam create-role \
  --role-name s3-access-role \
  --assume-role-policy-document file://trust-policy.json
```

## Step 5 - Attach an S3 Permission Policy

```bash
# Create an S3 policy
cat > s3-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-app-bucket", "arn:aws:s3:::my-app-bucket/*"]
    }
  ]
}
EOF

aws --endpoint-url http://rgw.example.com:7480 \
  iam put-role-policy \
  --role-name s3-access-role \
  --policy-name s3-access-policy \
  --policy-document file://s3-policy.json
```

## Step 6 - Exchange a Keycloak Token for S3 Credentials

```python
import requests
import boto3

# Step 1: Get a Keycloak JWT token
token_resp = requests.post(
    "https://keycloak.example.com/auth/realms/myrealm/protocol/openid-connect/token",
    data={
        "client_id": "ceph-rgw",
        "client_secret": "my-client-secret",
        "grant_type": "client_credentials"
    }
)
jwt_token = token_resp.json()["access_token"]

# Step 2: Assume role with the JWT token
sts_client = boto3.client(
    "sts",
    endpoint_url="http://rgw.example.com:7480",
    aws_access_key_id="", aws_secret_access_key=""
)
creds = sts_client.assume_role_with_web_identity(
    RoleArn="arn:aws:iam:::role/s3-access-role",
    RoleSessionName="app-session",
    WebIdentityToken=jwt_token
)
print(creds["Credentials"])
```

## Summary

Keycloak OIDC authentication for Ceph RGW enables applications to exchange JWT tokens for temporary S3 credentials via `AssumeRoleWithWebIdentity`. This eliminates long-lived access keys by using Keycloak's identity federation. The flow requires enabling STS in RGW, registering the OIDC provider, creating IAM roles, and attaching S3 permission policies before applications can seamlessly authenticate.
