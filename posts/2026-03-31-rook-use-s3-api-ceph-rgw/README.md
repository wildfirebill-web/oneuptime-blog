# How to Use the S3 API with Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, S3, API, Object Storage

Description: Learn how to use the S3 API with Ceph RGW for object storage operations including bucket creation, object upload, download, and access control.

---

## Overview

Ceph RGW implements the Amazon S3 API, making it compatible with any S3 client including AWS CLI, boto3, and s3cmd. This guide covers common S3 operations against a Ceph RGW endpoint using both the CLI and Python SDK.

## Prerequisites

Ensure you have:
- A running RGW instance with an accessible endpoint
- A user account with S3 credentials

```bash
# Get the RGW endpoint in a Rook cluster
kubectl -n rook-ceph get svc rook-ceph-rgw-my-store -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Retrieve user credentials from the secret
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-myuser \
  -o jsonpath='{.data.AccessKey}' | base64 -d
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-myuser \
  -o jsonpath='{.data.SecretKey}' | base64 -d
```

## Using AWS CLI with RGW

Configure the AWS CLI for Ceph RGW:

```bash
# Configure credentials
aws configure set aws_access_key_id MYACCESSKEY
aws configure set aws_secret_access_key MYSECRETKEY
aws configure set region us-east-1

# Set the RGW endpoint for each command
export RGW_ENDPOINT=http://rgw.example.com:80
```

## Bucket Operations

```bash
# Create a bucket
aws --endpoint-url $RGW_ENDPOINT s3 mb s3://mybucket

# List buckets
aws --endpoint-url $RGW_ENDPOINT s3 ls

# Enable versioning on a bucket
aws --endpoint-url $RGW_ENDPOINT s3api put-bucket-versioning \
  --bucket mybucket \
  --versioning-configuration Status=Enabled
```

## Object Operations

```bash
# Upload an object
aws --endpoint-url $RGW_ENDPOINT s3 cp file.txt s3://mybucket/file.txt

# Upload a directory recursively
aws --endpoint-url $RGW_ENDPOINT s3 sync ./mydir s3://mybucket/mydir/

# Download an object
aws --endpoint-url $RGW_ENDPOINT s3 cp s3://mybucket/file.txt ./file-copy.txt

# List objects
aws --endpoint-url $RGW_ENDPOINT s3 ls s3://mybucket --recursive

# Delete an object
aws --endpoint-url $RGW_ENDPOINT s3 rm s3://mybucket/file.txt
```

## Access Control

```bash
# Apply a bucket policy (public read)
cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::mybucket/*"
    }
  ]
}
EOF

aws --endpoint-url $RGW_ENDPOINT s3api put-bucket-policy \
  --bucket mybucket \
  --policy file://bucket-policy.json
```

## Using boto3 with Ceph RGW

```python
import boto3
from botocore.config import Config

s3 = boto3.client(
    's3',
    endpoint_url='http://rgw.example.com:80',
    aws_access_key_id='MYACCESSKEY',
    aws_secret_access_key='MYSECRETKEY',
    config=Config(signature_version='s3v4'),
    region_name='us-east-1'
)

# Create bucket
s3.create_bucket(Bucket='mybucket')

# Upload object
s3.upload_file('localfile.txt', 'mybucket', 'remotefile.txt')

# Download object
s3.download_file('mybucket', 'remotefile.txt', 'downloaded.txt')

# List objects
response = s3.list_objects_v2(Bucket='mybucket')
for obj in response.get('Contents', []):
    print(obj['Key'], obj['Size'])
```

## Multipart Upload for Large Files

```bash
# Use AWS CLI which handles multipart automatically
aws --endpoint-url $RGW_ENDPOINT \
  s3 cp large-file.iso s3://mybucket/large-file.iso \
  --expected-size 10737418240
```

## Summary

Ceph RGW provides full S3 API compatibility, allowing you to use any S3 client by pointing it at the RGW endpoint with your user credentials. The AWS CLI, boto3, and s3cmd all work with RGW with minor configuration changes. Use `--endpoint-url` with the CLI or `endpoint_url` in boto3 to redirect requests from AWS to your Ceph cluster.
