# How to Use S3 Object Tagging in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Tagging, Metadata, Storage

Description: Use S3 object and bucket tagging in Ceph RGW to add metadata for cost allocation, lifecycle filtering, access control, and data classification workflows.

---

## Overview

S3 object tagging lets you attach up to 10 key-value pairs to any object. Ceph RGW fully supports the tagging API, enabling you to classify objects, drive lifecycle policies based on tags, and add custom metadata without modifying object content.

## Add Tags When Uploading

```bash
aws s3 cp /tmp/report.pdf s3://my-bucket/reports/report.pdf \
  --tagging "department=finance&year=2026&confidential=true" \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Set Tags on Existing Objects

```bash
aws s3api put-object-tagging \
  --bucket my-bucket \
  --key reports/report.pdf \
  --tagging '{
    "TagSet": [
      {"Key": "department", "Value": "finance"},
      {"Key": "year", "Value": "2026"},
      {"Key": "retention", "Value": "7years"}
    ]
  }' \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Get Object Tags

```bash
aws s3api get-object-tagging \
  --bucket my-bucket \
  --key reports/report.pdf \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Delete Tags from an Object

```bash
aws s3api delete-object-tagging \
  --bucket my-bucket \
  --key reports/report.pdf \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Bucket Tagging

Tag a bucket:

```bash
aws s3api put-bucket-tagging \
  --bucket my-bucket \
  --tagging '{
    "TagSet": [
      {"Key": "team", "Value": "data-engineering"},
      {"Key": "env", "Value": "production"},
      {"Key": "cost-center", "Value": "DE-001"}
    ]
  }' \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph

aws s3api get-bucket-tagging \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Tag-Based Lifecycle Rules

Use tags in lifecycle rules to expire specific tagged objects:

```json
{
  "Rules": [
    {
      "ID": "expire-temp-objects",
      "Status": "Enabled",
      "Filter": {
        "Tag": {
          "Key": "temp",
          "Value": "true"
        }
      },
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
```

## Bulk Tag Objects with boto3

```python
paginator = s3.get_paginator("list_objects_v2")
for page in paginator.paginate(Bucket="my-bucket", Prefix="logs/2025/"):
    for obj in page.get("Contents", []):
        s3.put_object_tagging(
            Bucket="my-bucket",
            Key=obj["Key"],
            Tagging={
                "TagSet": [
                    {"Key": "year", "Value": "2025"},
                    {"Key": "archived", "Value": "true"},
                ]
            },
        )
```

## Summary

Ceph RGW's full S3 tagging support enables rich object classification without changing object content. Tags drive lifecycle policies, support data governance workflows, and serve as searchable metadata for custom analytics, making them a versatile tool for managing large object stores.
