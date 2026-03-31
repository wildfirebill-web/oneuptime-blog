# How to Configure S3 Bucket Lifecycle Policies in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Lifecycle, Storage, DevOps

Description: Learn how to configure S3-compatible bucket lifecycle policies in Ceph RGW to automatically expire objects, transition storage classes, and clean up incomplete uploads.

---

## Overview

Ceph RGW supports S3 bucket lifecycle policies, allowing you to automate object expiration, abort incomplete multipart uploads, and manage storage costs. These policies run periodically on the cluster using Ceph's lifecycle management daemon.

## Enable Lifecycle Processing

Ensure the lifecycle processing interval is set in your Ceph config:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_lifecycle_work_time "00:00-06:00"
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_lc_max_worker 3
```

## Set a Lifecycle Policy via AWS CLI

Create a lifecycle policy JSON file `lifecycle.json`:

```json
{
  "Rules": [
    {
      "ID": "expire-old-logs",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Expiration": {
        "Days": 30
      }
    },
    {
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

Apply the policy:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Set Lifecycle with boto3

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    "s3",
    endpoint_url="http://rook-ceph-rgw-my-store.rook-ceph:80",
    aws_access_key_id="myaccesskey",
    aws_secret_access_key="mysecretkey",
    config=Config(s3={"addressing_style": "path"}),
)

lifecycle = {
    "Rules": [
        {
            "ID": "expire-tmp",
            "Status": "Enabled",
            "Filter": {"Prefix": "tmp/"},
            "Expiration": {"Days": 1},
        }
    ]
}

s3.put_bucket_lifecycle_configuration(
    Bucket="my-bucket",
    LifecycleConfiguration=lifecycle,
)
```

## Retrieve Current Lifecycle Policy

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Delete a Lifecycle Policy

```bash
aws s3api delete-bucket-lifecycle \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Verify Lifecycle Processing

Check the Ceph RGW logs to confirm lifecycle processing is running:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-rgw-my-store \
  | grep -i lifecycle | tail -20
```

## Summary

Ceph RGW's S3-compatible lifecycle policy engine lets you automate data retention and storage management without custom scripts. By configuring expiration and abort rules, you can keep buckets lean and reduce storage consumption over time with zero manual intervention.
