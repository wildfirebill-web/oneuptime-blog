# How to Set Up S3 Bucket Versioning in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Versioning, Storage, Data Protection

Description: Enable and manage S3 bucket versioning on Ceph RGW to protect against accidental object deletion and maintain a full history of object changes.

---

## Overview

Ceph RGW supports S3-compatible bucket versioning. When enabled, every PUT overwrites result in a new version rather than replacing the existing object, and DELETE operations create a delete marker. This protects against accidental data loss and enables recovery of previous object states.

## Enable Versioning on a Bucket

Using AWS CLI:

```bash
aws s3api put-bucket-versioning \
  --bucket my-versioned-bucket \
  --versioning-configuration Status=Enabled \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

Check the versioning status:

```bash
aws s3api get-bucket-versioning \
  --bucket my-versioned-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Upload Multiple Versions of an Object

```bash
echo "version 1" > /tmp/myfile.txt
aws s3 cp /tmp/myfile.txt s3://my-versioned-bucket/myfile.txt \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph

echo "version 2" > /tmp/myfile.txt
aws s3 cp /tmp/myfile.txt s3://my-versioned-bucket/myfile.txt \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## List All Versions of an Object

```bash
aws s3api list-object-versions \
  --bucket my-versioned-bucket \
  --prefix myfile.txt \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Retrieve a Specific Version

```bash
aws s3api get-object \
  --bucket my-versioned-bucket \
  --key myfile.txt \
  --version-id <VERSION_ID> \
  /tmp/restored-v1.txt \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Delete a Specific Version

```bash
aws s3api delete-object \
  --bucket my-versioned-bucket \
  --key myfile.txt \
  --version-id <VERSION_ID> \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Suspend Versioning

Suspending preserves existing versions but stops creating new ones:

```bash
aws s3api put-bucket-versioning \
  --bucket my-versioned-bucket \
  --versioning-configuration Status=Suspended \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Combine with Lifecycle Policy

Automatically expire old versions after 90 days:

```json
{
  "Rules": [
    {
      "ID": "expire-old-versions",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```

## Summary

Ceph RGW's S3 bucket versioning provides object history and protection against accidental deletion, working identically to AWS S3. Combined with lifecycle policies that expire old versions, you can maintain a safe version history without unbounded storage growth.
