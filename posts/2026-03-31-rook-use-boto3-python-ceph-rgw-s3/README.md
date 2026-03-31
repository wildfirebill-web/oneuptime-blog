# How to Use boto3 (Python) with Ceph RGW S3

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Python, Boto3, S3, RGW, Storage

Description: Use the Python boto3 library to interact with Ceph RGW S3-compatible API for bucket management, object operations, and presigned URLs in your applications.

---

## Overview

`boto3` is the official AWS SDK for Python. By passing a custom `endpoint_url` and disabling SSL for development, you can use it to interact with Ceph RGW exactly as you would with AWS S3. This makes it easy to write Python applications that work against your on-premises Ceph cluster.

## Install boto3

```bash
pip install boto3
```

## Create a boto3 Session for Ceph

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    "s3",
    endpoint_url="http://rook-ceph-rgw-my-store.rook-ceph:80",
    aws_access_key_id="myaccesskey",
    aws_secret_access_key="mysecretkey",
    region_name="us-east-1",
    config=Config(signature_version="s3v4", s3={"addressing_style": "path"}),
)
```

## Bucket Operations

Create a bucket:

```python
s3.create_bucket(Bucket="my-python-bucket")
```

List all buckets:

```python
response = s3.list_buckets()
for bucket in response["Buckets"]:
    print(bucket["Name"])
```

Delete a bucket (must be empty):

```python
s3.delete_bucket(Bucket="my-python-bucket")
```

## Object Operations

Upload a file:

```python
s3.upload_file(
    Filename="/tmp/data.json",
    Bucket="my-python-bucket",
    Key="data/data.json",
)
```

Upload from memory:

```python
import json

data = {"key": "value", "count": 42}
s3.put_object(
    Bucket="my-python-bucket",
    Key="config/settings.json",
    Body=json.dumps(data),
    ContentType="application/json",
)
```

Download a file:

```python
s3.download_file(
    Bucket="my-python-bucket",
    Key="data/data.json",
    Filename="/tmp/downloaded.json",
)
```

List objects in a bucket:

```python
paginator = s3.get_paginator("list_objects_v2")
for page in paginator.paginate(Bucket="my-python-bucket", Prefix="data/"):
    for obj in page.get("Contents", []):
        print(obj["Key"], obj["Size"])
```

## Generate a Presigned URL

```python
url = s3.generate_presigned_url(
    ClientMethod="get_object",
    Params={"Bucket": "my-python-bucket", "Key": "data/data.json"},
    ExpiresIn=3600,
)
print(url)
```

## Copy Objects

```python
s3.copy_object(
    CopySource={"Bucket": "my-python-bucket", "Key": "data/data.json"},
    Bucket="my-python-bucket",
    Key="backup/data.json",
)
```

## Error Handling

```python
from botocore.exceptions import ClientError

try:
    s3.head_object(Bucket="my-python-bucket", Key="nonexistent.txt")
except ClientError as e:
    if e.response["Error"]["Code"] == "404":
        print("Object does not exist")
    else:
        raise
```

## Summary

`boto3` works transparently with Ceph RGW by specifying a custom `endpoint_url` and path-style addressing. All standard S3 operations are supported, making it easy to build Python applications that run against both AWS S3 and on-premises Ceph with minimal configuration changes.
