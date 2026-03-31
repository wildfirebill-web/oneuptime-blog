# How to Set Up Bucket Versioning in Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Versioning, S3, Data Protection, Kubernetes

Description: Learn how to enable and manage S3 bucket versioning in the Rook Ceph Object Store to protect against accidental deletions and overwrite objects.

---

## Overview

Bucket versioning in the Rook Ceph Object Store allows you to preserve multiple versions of an object in the same bucket. When versioning is enabled, every overwrite or deletion creates a new version rather than permanently replacing or deleting the object, protecting against accidental data loss.

## Enabling Versioning on a Bucket

Use the AWS CLI to enable versioning on a bucket:

```bash
ENDPOINT=http://$(kubectl -n rook-ceph get svc rook-ceph-rgw-my-store \
  -o jsonpath='{.spec.clusterIP}'):80

aws --endpoint-url $ENDPOINT \
  s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled
```

Verify versioning is enabled:

```bash
aws --endpoint-url $ENDPOINT \
  s3api get-bucket-versioning \
  --bucket my-bucket
```

Expected output:

```json
{
    "Status": "Enabled"
}
```

## Uploading and Listing Object Versions

Upload multiple versions of the same object:

```bash
echo "version 1" > file.txt
aws --endpoint-url $ENDPOINT s3 cp file.txt s3://my-bucket/file.txt

echo "version 2" > file.txt
aws --endpoint-url $ENDPOINT s3 cp file.txt s3://my-bucket/file.txt

echo "version 3" > file.txt
aws --endpoint-url $ENDPOINT s3 cp file.txt s3://my-bucket/file.txt
```

List all versions of the object:

```bash
aws --endpoint-url $ENDPOINT \
  s3api list-object-versions \
  --bucket my-bucket \
  --prefix file.txt
```

Output showing three versions:

```json
{
    "Versions": [
        {
            "ETag": "...",
            "Key": "file.txt",
            "VersionId": "abc123",
            "IsLatest": true,
            "LastModified": "2026-03-31T10:00:00Z"
        },
        {
            "ETag": "...",
            "Key": "file.txt",
            "VersionId": "def456",
            "IsLatest": false,
            "LastModified": "2026-03-31T09:55:00Z"
        }
    ]
}
```

## Retrieving a Specific Version

Download a specific version using its version ID:

```bash
aws --endpoint-url $ENDPOINT \
  s3api get-object \
  --bucket my-bucket \
  --key file.txt \
  --version-id def456 \
  output-file.txt
```

## Deleting a Specific Version

Delete a specific version permanently:

```bash
aws --endpoint-url $ENDPOINT \
  s3api delete-object \
  --bucket my-bucket \
  --key file.txt \
  --version-id abc123
```

When you delete without a version ID in a versioned bucket, a delete marker is created:

```bash
aws --endpoint-url $ENDPOINT \
  s3 rm s3://my-bucket/file.txt
```

## Suspending Versioning

Suspend versioning to stop creating new versions (existing versions are retained):

```bash
aws --endpoint-url $ENDPOINT \
  s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Suspended
```

## Combining Versioning with Lifecycle Policies

Use lifecycle rules to manage version accumulation:

```json
{
  "Rules": [
    {
      "ID": "cleanup-old-versions",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30,
        "NewerNoncurrentVersions": 5
      }
    }
  ]
}
```

Apply the lifecycle policy:

```bash
aws --endpoint-url $ENDPOINT \
  s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

## Summary

Bucket versioning in the Rook Object Store is enabled via the standard S3 API and provides protection against accidental deletions and overwrites by maintaining multiple versions of each object. You can retrieve, list, and delete specific versions using version IDs. Combining versioning with lifecycle policies allows you to automatically clean up old versions while retaining a configurable number of recent versions.
