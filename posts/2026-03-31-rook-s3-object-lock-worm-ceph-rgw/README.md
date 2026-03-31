# How to Configure S3 Object Lock (WORM) in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Object Lock, WORM, Compliance

Description: Enable S3 Object Lock on Ceph RGW buckets to enforce Write-Once-Read-Many (WORM) retention for compliance, audit logs, and immutable backup storage.

---

## Overview

S3 Object Lock provides WORM (Write-Once-Read-Many) protection for objects. Once an object is locked, it cannot be deleted or overwritten until its retention period expires. Ceph RGW supports Object Lock in both Compliance and Governance modes, making it suitable for regulatory compliance scenarios.

## Enable Object Lock at Bucket Creation

Object Lock must be enabled when the bucket is created (it cannot be enabled later):

```bash
aws s3api create-bucket \
  --bucket compliance-bucket \
  --object-lock-enabled-for-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

Versioning is automatically enabled when Object Lock is enabled.

## Set Default Retention on a Bucket

Apply a bucket-level default retention policy:

```bash
aws s3api put-object-lock-configuration \
  --bucket compliance-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 365
      }
    }
  }' \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Upload a Locked Object

Upload with an explicit retention date:

```bash
aws s3api put-object \
  --bucket compliance-bucket \
  --key audit-log-2026-03-31.json \
  --body /tmp/audit-log.json \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date "2027-03-31T00:00:00Z" \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Use Governance Mode for Testing

Governance mode allows privileged users to override retention:

```bash
aws s3api put-object \
  --bucket compliance-bucket \
  --key test-object.txt \
  --body /tmp/test.txt \
  --object-lock-mode GOVERNANCE \
  --object-lock-retain-until-date "2026-12-31T00:00:00Z" \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

Override governance retention (requires special permission):

```bash
aws s3api delete-object \
  --bucket compliance-bucket \
  --key test-object.txt \
  --bypass-governance-retention \
  --version-id <VERSION_ID> \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Set a Legal Hold

Legal holds block deletion indefinitely regardless of retention:

```bash
aws s3api put-object-legal-hold \
  --bucket compliance-bucket \
  --key audit-log-2026-03-31.json \
  --legal-hold '{"Status": "ON"}' \
  --version-id <VERSION_ID> \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Get Object Retention Info

```bash
aws s3api get-object-retention \
  --bucket compliance-bucket \
  --key audit-log-2026-03-31.json \
  --version-id <VERSION_ID> \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Summary

Ceph RGW's S3 Object Lock implementation supports both Compliance and Governance WORM modes, making it suitable for financial records, audit logs, and regulated data. Enabling Object Lock at bucket creation is the only requirement before you can start protecting objects with immutable retention policies.
