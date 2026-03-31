# How to Configure Lifecycle Policies for Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Lifecycle, S3, Data Management, Kubernetes

Description: Learn how to configure S3-compatible lifecycle policies on Rook Ceph Object Store buckets to automatically expire, transition, or delete objects.

---

## Overview

Lifecycle policies in the Rook Ceph Object Store allow you to automate data management tasks on S3 buckets. You can automatically expire old objects, delete incomplete multipart uploads, and manage object versions to control storage costs and enforce data retention policies.

Rook's RGW implements the S3 Lifecycle API, so standard S3 lifecycle configurations work directly with your Ceph object store.

## Prerequisites

Ensure you have a running CephObjectStore and a bucket. Get the RGW endpoint and credentials:

```bash
ENDPOINT=http://$(kubectl -n rook-ceph get svc rook-ceph-rgw-my-store -o jsonpath='{.spec.clusterIP}'):80
ACCESS_KEY=$(kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user \
  -o jsonpath='{.data.AccessKey}' | base64 --decode)
SECRET_KEY=$(kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user \
  -o jsonpath='{.data.SecretKey}' | base64 --decode)
```

## Creating a Lifecycle Policy with AWS CLI

Create a lifecycle configuration JSON file:

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
      "ID": "cleanup-incomplete-uploads",
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

Apply the policy to a bucket:

```bash
aws --endpoint-url $ENDPOINT \
  --access-key $ACCESS_KEY \
  --secret-key $SECRET_KEY \
  s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

## Viewing Existing Lifecycle Policies

Retrieve the current lifecycle configuration:

```bash
aws --endpoint-url $ENDPOINT \
  s3api get-bucket-lifecycle-configuration \
  --bucket my-bucket
```

## Advanced Lifecycle Rules

### Expiring Objects by Age and Size

```json
{
  "Rules": [
    {
      "ID": "expire-large-old-objects",
      "Status": "Enabled",
      "Filter": {
        "And": {
          "Prefix": "uploads/",
          "ObjectSizeGreaterThan": 104857600
        }
      },
      "Expiration": {
        "Days": 90
      }
    }
  ]
}
```

### Managing Versioned Object Expiration

For buckets with versioning enabled:

```json
{
  "Rules": [
    {
      "ID": "expire-old-versions",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      },
      "ExpiredObjectDeleteMarker": true
    }
  ]
}
```

## Configuring Lifecycle Processing Interval

Ceph processes lifecycle rules periodically. Adjust the processing interval via a config override:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [global]
    rgw lifecycle work time = 00:00-06:00
    rgw lc debug interval = 10
```

This restricts lifecycle processing to off-peak hours (midnight to 6 AM).

## Removing a Lifecycle Policy

Delete all lifecycle rules from a bucket:

```bash
aws --endpoint-url $ENDPOINT \
  s3api delete-bucket-lifecycle \
  --bucket my-bucket
```

## Monitoring Lifecycle Execution

Check RGW logs for lifecycle processing activity:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-rgw-my-store-a | grep -i lifecycle
```

## Summary

Configuring lifecycle policies for the Rook Object Store uses the standard S3 lifecycle API supported by Ceph RGW. You can define rules to expire objects by age, clean up incomplete multipart uploads, and manage non-current versions in versioned buckets. The lifecycle processor runs on a configurable schedule and processes all buckets with active rules, making it a powerful tool for automated data retention management.
