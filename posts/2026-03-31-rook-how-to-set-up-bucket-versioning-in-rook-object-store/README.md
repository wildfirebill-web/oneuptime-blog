# How to Set Up Bucket Versioning in Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Bucket Versioning, Object Storage, S3, Kubernetes

Description: Enable S3-compatible bucket versioning on Rook Object Store to preserve, retrieve, and restore every version of objects stored in your buckets.

---

## Overview

Bucket versioning in Ceph RGW works like S3 versioning: when enabled, every PUT to an existing key creates a new version instead of overwriting it. Deletes also create a delete marker rather than permanently removing the data. This provides protection against accidental deletion and enables point-in-time recovery.

## Enable Versioning on a Bucket

Using the AWS CLI pointed at your RGW endpoint:

```bash
export RGW_ENDPOINT=http://rook-ceph-rgw-my-store.rook-ceph.svc.cluster.local:80

aws s3api put-bucket-versioning \
  --endpoint-url $RGW_ENDPOINT \
  --bucket my-versioned-bucket \
  --versioning-configuration Status=Enabled
```

## Check Versioning Status

```bash
aws s3api get-bucket-versioning \
  --endpoint-url $RGW_ENDPOINT \
  --bucket my-versioned-bucket
```

Output:

```json
{
    "Status": "Enabled"
}
```

## Working with Versioned Objects

### Upload Multiple Versions

```bash
echo "version 1" > file.txt
aws s3 cp file.txt s3://my-versioned-bucket/file.txt \
  --endpoint-url $RGW_ENDPOINT

echo "version 2" > file.txt
aws s3 cp file.txt s3://my-versioned-bucket/file.txt \
  --endpoint-url $RGW_ENDPOINT
```

### List All Versions

```bash
aws s3api list-object-versions \
  --endpoint-url $RGW_ENDPOINT \
  --bucket my-versioned-bucket \
  --prefix file.txt
```

Output shows version IDs:

```json
{
    "Versions": [
        {
            "Key": "file.txt",
            "VersionId": "abc123",
            "IsLatest": true,
            "LastModified": "2026-03-31T10:00:00Z",
            "Size": 10
        },
        {
            "Key": "file.txt",
            "VersionId": "xyz789",
            "IsLatest": false,
            "LastModified": "2026-03-31T09:00:00Z",
            "Size": 10
        }
    ]
}
```

### Retrieve a Specific Version

```bash
aws s3api get-object \
  --endpoint-url $RGW_ENDPOINT \
  --bucket my-versioned-bucket \
  --key file.txt \
  --version-id xyz789 \
  recovered-version.txt
```

### Delete a Specific Version

```bash
aws s3api delete-object \
  --endpoint-url $RGW_ENDPOINT \
  --bucket my-versioned-bucket \
  --key file.txt \
  --version-id abc123
```

### Suspend Versioning

Suspending versioning stops creating new versions but preserves existing ones:

```bash
aws s3api put-bucket-versioning \
  --endpoint-url $RGW_ENDPOINT \
  --bucket my-versioned-bucket \
  --versioning-configuration Status=Suspended
```

## Enable Versioning via ObjectBucketClaim Annotation

You can enable versioning automatically when creating a bucket through Rook's OBC:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-versioned-obc
  namespace: default
  annotations:
    ceph.rook.io/bucket-versioning: "true"
spec:
  generateBucketName: my-versioned-bucket
  storageClassName: rook-ceph-bucket
```

## Combine with Lifecycle Policies

Use lifecycle policies to automatically clean up old versions and control storage growth:

```json
{
  "Rules": [
    {
      "ID": "cleanup-old-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30,
        "NewerNoncurrentVersions": 5
      }
    }
  ]
}
```

## Summary

Bucket versioning in Rook Object Store enables S3-compatible object version history, protecting against accidental deletion and enabling point-in-time recovery. Enable it via the `put-bucket-versioning` API call or through OBC annotations. Always pair versioning with lifecycle policies to automatically expire old versions and prevent unbounded storage growth.
