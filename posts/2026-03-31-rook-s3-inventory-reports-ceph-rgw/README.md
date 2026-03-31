# How to Configure S3 Inventory Reports in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Inventory, Reporting, Storage

Description: Set up S3 inventory reports in Ceph RGW to generate scheduled CSV or ORC listings of bucket objects with metadata, useful for auditing and data management.

---

## Overview

S3 Inventory Reports provide scheduled flat-file listings of all objects in a bucket, including metadata like size, ETag, encryption status, and replication status. Ceph RGW supports this API, allowing you to generate automated inventory snapshots for compliance, auditing, and data cataloging.

## Prerequisites

- Ceph RGW with inventory support enabled (Ceph Quincy or later)
- A destination bucket for inventory reports
- Appropriate IAM/RGW user permissions

## Create a Destination Bucket

```bash
aws s3 mb s3://inventory-reports \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Configure an Inventory Report

Create `inventory-config.json`:

```json
{
  "Destination": {
    "S3BucketDestination": {
      "Bucket": "arn:aws:s3:::inventory-reports",
      "Format": "CSV",
      "Prefix": "my-bucket-inventory"
    }
  },
  "IsEnabled": true,
  "Id": "weekly-inventory",
  "IncludedObjectVersions": "Current",
  "Schedule": {
    "Frequency": "Weekly"
  },
  "OptionalFields": [
    "Size",
    "LastModifiedDate",
    "ETag",
    "StorageClass",
    "IsMultipartUploaded",
    "ReplicationStatus",
    "EncryptionStatus"
  ]
}
```

Apply the inventory configuration:

```bash
aws s3api put-bucket-inventory-configuration \
  --bucket my-bucket \
  --id weekly-inventory \
  --inventory-configuration file://inventory-config.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Retrieve Current Inventory Configuration

```bash
aws s3api get-bucket-inventory-configuration \
  --bucket my-bucket \
  --id weekly-inventory \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

List all inventory configurations:

```bash
aws s3api list-bucket-inventory-configurations \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Read an Inventory Report

After the first inventory runs, reports appear in the destination bucket:

```bash
aws s3 ls s3://inventory-reports/my-bucket-inventory/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph --recursive

aws s3 cp s3://inventory-reports/my-bucket-inventory/data/inventory.csv.gz /tmp/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph

gunzip /tmp/inventory.csv.gz
head /tmp/inventory.csv
```

Sample CSV output:

```text
"my-bucket","uploads/data.json","2026-03-31T10:00:00.000Z","1024","abc123","STANDARD","false","NOT-REPLICATED","NOT-SSE"
"my-bucket","logs/app.log","2026-03-31T09:00:00.000Z","4096","def456","STANDARD","false","NOT-REPLICATED","NOT-SSE"
```

## Delete an Inventory Configuration

```bash
aws s3api delete-bucket-inventory-configuration \
  --bucket my-bucket \
  --id weekly-inventory \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Summary

Ceph RGW inventory reports provide automated, scheduled object listings in CSV format that are ideal for auditing, compliance checks, and storage analytics. The reports land in a destination bucket and can be processed with standard data tools, giving you a reliable snapshot of bucket contents without running expensive LIST operations.
