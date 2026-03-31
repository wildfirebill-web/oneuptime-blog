# How to Use S3 Batch Operations with Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Batch Operations, Automation, Storage

Description: Perform bulk S3 operations on millions of objects in Ceph RGW using S3 Batch Operations patterns with boto3 and AWS CLI for efficient large-scale data management.

---

## Overview

AWS S3 Batch Operations is a managed service for running bulk actions on millions of objects. Ceph RGW does not implement the Batch Operations Job API directly, but you can implement equivalent bulk operations using the S3 API, inventory reports, and scripted workflows. This guide covers practical patterns for batch processing objects in Ceph.

## Pattern 1 - Batch Copy with boto3

Copy all objects with a specific prefix to another bucket:

```python
import boto3
from botocore.client import Config
from concurrent.futures import ThreadPoolExecutor

s3 = boto3.client(
    "s3",
    endpoint_url="http://rook-ceph-rgw-my-store.rook-ceph:80",
    aws_access_key_id="myaccesskey",
    aws_secret_access_key="mysecretkey",
    config=Config(s3={"addressing_style": "path"}),
)

def copy_object(key):
    s3.copy_object(
        CopySource={"Bucket": "source-bucket", "Key": key},
        Bucket="dest-bucket",
        Key=key,
    )
    print(f"Copied: {key}")

paginator = s3.get_paginator("list_objects_v2")
keys = []
for page in paginator.paginate(Bucket="source-bucket", Prefix="data/"):
    for obj in page.get("Contents", []):
        keys.append(obj["Key"])

with ThreadPoolExecutor(max_workers=20) as executor:
    executor.map(copy_object, keys)

print(f"Copied {len(keys)} objects")
```

## Pattern 2 - Batch Delete

Delete all objects matching a tag or prefix:

```python
paginator = s3.get_paginator("list_objects_v2")
to_delete = []

for page in paginator.paginate(Bucket="my-bucket", Prefix="tmp/"):
    for obj in page.get("Contents", []):
        to_delete.append({"Key": obj["Key"]})
        if len(to_delete) == 1000:
            s3.delete_objects(
                Bucket="my-bucket",
                Delete={"Objects": to_delete, "Quiet": True},
            )
            print(f"Deleted batch of {len(to_delete)}")
            to_delete = []

if to_delete:
    s3.delete_objects(
        Bucket="my-bucket",
        Delete={"Objects": to_delete, "Quiet": True},
    )
    print(f"Deleted final batch of {len(to_delete)}")
```

## Pattern 3 - Batch Tag Update

Add tags to all objects in a prefix:

```python
paginator = s3.get_paginator("list_objects_v2")
updated = 0

for page in paginator.paginate(Bucket="my-bucket", Prefix="archive/"):
    for obj in page.get("Contents", []):
        s3.put_object_tagging(
            Bucket="my-bucket",
            Key=obj["Key"],
            Tagging={
                "TagSet": [
                    {"Key": "status", "Value": "archived"},
                    {"Key": "retention", "Value": "7years"},
                ]
            },
        )
        updated += 1

print(f"Tagged {updated} objects")
```

## Pattern 4 - Batch ACL Change via Shell

Using AWS CLI with xargs for parallel processing:

```bash
aws s3api list-objects-v2 \
  --bucket my-bucket \
  --prefix public/ \
  --query 'Contents[].Key' \
  --output text \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph | \
tr '\t' '\n' | \
xargs -P 10 -I {} aws s3api put-object-acl \
  --bucket my-bucket \
  --key {} \
  --acl public-read \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Summary

While Ceph RGW does not expose a dedicated Batch Operations Job API, all the underlying S3 APIs are available for building equivalent bulk workflows. Using Python's `ThreadPoolExecutor` or shell parallelism with `xargs`, you can efficiently process millions of objects in Ceph at high concurrency.
