# How to Use Python Bindings for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Python, SDK, Object Storage

Description: Learn how to use Python bindings and libraries for Ceph RGW, including boto3 for S3 operations and the rgwadmin library for administrative tasks.

---

## Overview

Python is a primary language for interacting with Ceph RGW, both for S3 object storage operations and administrative tasks. The `boto3` library provides full S3 API access, while `python-rgwadmin` and direct requests to the Admin Ops API enable programmatic administration. This guide covers both patterns with practical examples.

## Setting Up boto3 for Ceph RGW

```bash
pip install boto3
```

```python
import boto3
from botocore.config import Config

def get_s3_client(endpoint, access_key, secret_key):
    return boto3.client(
        's3',
        endpoint_url=endpoint,
        aws_access_key_id=access_key,
        aws_secret_access_key=secret_key,
        config=Config(
            signature_version='s3v4',
            retries={'max_attempts': 3, 'mode': 'adaptive'}
        ),
        region_name='us-east-1'
    )

s3 = get_s3_client(
    'http://rgw.example.com:80',
    'ACCESSKEY',
    'SECRETKEY'
)
```

## Bucket Operations with boto3

```python
# Create a bucket
s3.create_bucket(Bucket='my-python-bucket')

# Enable versioning
s3.put_bucket_versioning(
    Bucket='my-python-bucket',
    VersioningConfiguration={'Status': 'Enabled'}
)

# Set lifecycle rules
s3.put_bucket_lifecycle_configuration(
    Bucket='my-python-bucket',
    LifecycleConfiguration={
        'Rules': [{
            'ID': 'expire-old-objects',
            'Status': 'Enabled',
            'Expiration': {'Days': 30},
            'Filter': {'Prefix': 'tmp/'}
        }]
    }
)
```

## Object Operations with boto3

```python
import json

# Upload with metadata
s3.put_object(
    Bucket='my-python-bucket',
    Key='config.json',
    Body=json.dumps({'version': '1.0', 'env': 'prod'}),
    ContentType='application/json',
    Metadata={'author': 'alice', 'team': 'platform'}
)

# Download and parse
response = s3.get_object(Bucket='my-python-bucket', Key='config.json')
config = json.loads(response['Body'].read())

# Multipart upload for large files
def upload_large_file(bucket, key, filepath, chunk_size=50*1024*1024):
    mpu = s3.create_multipart_upload(Bucket=bucket, Key=key)
    upload_id = mpu['UploadId']
    parts = []

    with open(filepath, 'rb') as f:
        part_num = 1
        while chunk := f.read(chunk_size):
            response = s3.upload_part(
                Bucket=bucket, Key=key,
                PartNumber=part_num, UploadId=upload_id, Body=chunk
            )
            parts.append({'PartNumber': part_num, 'ETag': response['ETag']})
            part_num += 1

    s3.complete_multipart_upload(
        Bucket=bucket, Key=key, UploadId=upload_id,
        MultipartUpload={'Parts': parts}
    )
```

## Using rgwadmin for Administrative Tasks

```bash
pip install python-rgwadmin
```

```python
from rgwadmin import RGWAdmin

rgw = RGWAdmin(
    access_key='ADMINACCESSKEY',
    secret_key='ADMINSECRETKEY',
    server='rgw.example.com:80',
    secure=False
)

# Create a user
rgw.create_user(
    uid='alice',
    display_name='Alice Smith',
    email='alice@example.com'
)

# Get user info
user = rgw.get_user(uid='alice')
print(user)

# Set user quota
rgw.set_user_quota(
    uid='alice',
    max_size_kb=10*1024*1024,  # 10GB
    max_objects=1000000,
    quota_type='user'
)
rgw.enable_user_quota(uid='alice', quota_type='user')

# Get bucket stats
stats = rgw.get_bucket(bucket='mybucket', stats=True)
print(f"Bucket size: {stats['usage']['rgw.main']['size_kb']} KB")
```

## Generating Pre-Signed URLs

```python
from datetime import datetime

def generate_presigned_url(bucket, key, expiry_seconds=3600):
    url = s3.generate_presigned_url(
        'get_object',
        Params={'Bucket': bucket, 'Key': key},
        ExpiresIn=expiry_seconds
    )
    return url

url = generate_presigned_url('my-python-bucket', 'report.pdf')
print(f"Download URL (expires in 1 hour): {url}")
```

## Summary

Python provides two primary paths for working with Ceph RGW: `boto3` for S3-compatible object operations and `python-rgwadmin` for administrative tasks. Both libraries work seamlessly with Ceph by pointing at the RGW endpoint. Use boto3 for production application data access, multipart uploads, and lifecycle management, while rgwadmin simplifies user and quota administration without requiring the radosgw-admin CLI.
