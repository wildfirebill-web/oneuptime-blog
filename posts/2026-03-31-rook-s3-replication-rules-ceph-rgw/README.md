# How to Set Up S3 Replication Rules in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Replication, Disaster Recovery, Storage

Description: Configure S3-compatible replication rules in Ceph RGW to automatically replicate objects between buckets or zones for disaster recovery and data redundancy.

---

## Overview

Ceph RGW supports bucket replication between zones in a multisite setup, equivalent to S3 Cross-Region Replication. Objects written to a source bucket are automatically replicated to the destination bucket, enabling disaster recovery and geo-redundancy on-premises.

## Prerequisites

- At least two Ceph zones (zone1 and zone2) in the same zonegroup
- Both zones running RGW instances
- Versioning enabled on source and destination buckets

## Configure Multisite Zones

Set up a zonegroup with two zones:

```bash
# On the primary cluster
radosgw-admin realm create --rgw-realm=production --default
radosgw-admin zonegroup create --rgw-zonegroup=us --master --default
radosgw-admin zone create --rgw-zonegroup=us --rgw-zone=zone1 --master --default
radosgw-admin zone create --rgw-zonegroup=us --rgw-zone=zone2 \
  --endpoints=http://zone2-rgw.example.com:80
```

## Enable Versioning on Both Buckets

```bash
for bucket in source-bucket dest-bucket; do
  aws s3api put-bucket-versioning \
    --bucket $bucket \
    --versioning-configuration Status=Enabled \
    --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
    --profile ceph
done
```

## Set Replication Configuration

Create `replication.json`:

```json
{
  "Role": "arn:aws:iam:::role/replication-role",
  "Rules": [
    {
      "ID": "replicate-all",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::dest-bucket",
        "StorageClass": "STANDARD"
      },
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      }
    }
  ]
}
```

Apply:

```bash
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration file://replication.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Verify Replication Configuration

```bash
aws s3api get-bucket-replication \
  --bucket source-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Test Replication

Upload an object to the source bucket and verify it appears in the destination:

```bash
aws s3 cp /tmp/testfile.txt s3://source-bucket/testfile.txt \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph

# Check destination (zone2 endpoint)
aws s3 ls s3://dest-bucket/ \
  --endpoint-url http://zone2-rgw.example.com:80 \
  --profile ceph
```

## Monitor Sync Status

```bash
radosgw-admin sync status
radosgw-admin bucket sync status --bucket=source-bucket --source-zone=zone1
```

## Delete Replication Configuration

```bash
aws s3api delete-bucket-replication \
  --bucket source-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Summary

Ceph RGW's multisite replication provides S3-compatible disaster recovery between zones, automatically synchronizing objects from source to destination buckets. Combined with Rook's declarative zone management, you get a robust geo-redundant storage setup that requires no application changes.
