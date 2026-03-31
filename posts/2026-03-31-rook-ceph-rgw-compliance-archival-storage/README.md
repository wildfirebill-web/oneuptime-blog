# How to Use Ceph RGW for Compliance and Archival Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Compliance, Archival, Object Lock, S3, Governance

Description: Configure Ceph RGW with S3 Object Lock and lifecycle policies for WORM-compliant archival storage meeting regulatory requirements.

---

## Introduction

Many industries require immutable, tamper-proof storage for financial records, medical data, and audit logs. Ceph RGW supports S3 Object Lock, which enables WORM (Write Once Read Many) compliance. This guide shows how to configure RGW for regulatory archival use cases.

## Enabling Object Lock on a Bucket

Object Lock must be enabled at bucket creation time. First, ensure the RGW object store is running, then create a bucket with Object Lock:

```bash
aws s3api create-bucket \
  --bucket compliance-archive \
  --endpoint-url http://rook-ceph-rgw-store.rook-ceph:80 \
  --object-lock-enabled-for-bucket
```

Set a default retention policy:

```bash
aws s3api put-object-lock-configuration \
  --bucket compliance-archive \
  --endpoint-url http://rook-ceph-rgw-store.rook-ceph:80 \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Years": 7
      }
    }
  }'
```

## Uploading Compliant Objects

When uploading, you can override the default retention per object:

```bash
aws s3api put-object \
  --bucket compliance-archive \
  --key records/invoice-2026-001.pdf \
  --body invoice-2026-001.pdf \
  --endpoint-url http://rook-ceph-rgw-store.rook-ceph:80 \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date "2033-03-31T00:00:00Z"
```

Verify the lock is set:

```bash
aws s3api get-object-retention \
  --bucket compliance-archive \
  --key records/invoice-2026-001.pdf \
  --endpoint-url http://rook-ceph-rgw-store.rook-ceph:80
```

## Configuring Versioning for Audit Trails

Enable versioning alongside Object Lock to preserve every version:

```bash
aws s3api put-bucket-versioning \
  --bucket compliance-archive \
  --endpoint-url http://rook-ceph-rgw-store.rook-ceph:80 \
  --versioning-configuration Status=Enabled
```

## Setting Lifecycle Policies for Cost Management

After the compliance period expires, transition objects to erasure-coded pools:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket compliance-archive \
  --endpoint-url http://rook-ceph-rgw-store.rook-ceph:80 \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "move-to-deep-archive",
      "Status": "Enabled",
      "Filter": {"Prefix": "records/"},
      "Transitions": [
        {"Days": 365, "StorageClass": "GLACIER"}
      ]
    }]
  }'
```

## Enabling Access Logging for Audits

Configure bucket logging to capture every access event:

```bash
radosgw-admin bucket logging enable \
  --bucket compliance-archive \
  --target-bucket compliance-logs
```

Review logs:

```bash
aws s3 ls s3://compliance-logs/ \
  --endpoint-url http://rook-ceph-rgw-store.rook-ceph:80 \
  --recursive
```

## Summary

Ceph RGW with S3 Object Lock provides a self-hosted, WORM-compliant archival storage solution suitable for regulated industries. By combining compliance-mode object locking, versioning, lifecycle policies, and access logging, organizations can meet 7-year and longer retention requirements entirely on-premises without incurring cloud storage costs.
