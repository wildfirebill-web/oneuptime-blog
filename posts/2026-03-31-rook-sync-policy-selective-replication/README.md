# How to Configure Sync Policy for Selective Replication in RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Sync Policy, Replication, Object Storage

Description: Learn how to configure Ceph RGW sync policies for selective bucket replication, allowing you to replicate only specific buckets or prefixes to remote zones.

---

## What Is RGW Sync Policy

By default, Ceph RGW multisite replicates all data and metadata between zones. Sync policies allow fine-grained control:
- Replicate only specific buckets to specific zones
- Exclude certain buckets from replication
- Apply different replication rules per zone or zone group

## Step 1: Create a Group-Level Sync Policy

Sync policies use a group-based model. First create a policy group:

```bash
# Create a default policy group that replicas everything
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync group create \
  --group-id=default-group \
  --status=enabled

# Create a restrictive group for selective replication
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync group create \
  --group-id=selective-group \
  --status=allowed
```

## Step 2: Add Bucket-Level Sync Policy

Override replication for a specific bucket to control where it syncs:

```bash
# Allow bucket "critical-data" to replicate to us-west only
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync group create \
  --bucket=critical-data \
  --group-id=bucket-group \
  --status=enabled

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync group flow create \
  --bucket=critical-data \
  --group-id=bucket-group \
  --flow-id=replicate-to-west \
  --flow-type=symmetrical \
  --zones=us-east,us-west
```

## Step 3: Exclude a Bucket from Replication

```bash
# Create a "forbidden" policy for the test bucket
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync group create \
  --bucket=temp-data \
  --group-id=no-replicate \
  --status=forbidden
```

## Step 4: Add a Prefix Filter

Replicate only objects with a specific prefix:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync group pipe create \
  --bucket=app-data \
  --group-id=prefix-group \
  --pipe-id=replicate-logs \
  --source-zones='*' \
  --source-bucket=app-data \
  --dest-zones=us-west \
  --dest-bucket=app-data \
  --prefix=logs/
```

## Step 5: Verify Sync Policy

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync group get \
  --bucket=critical-data \
  --group-id=bucket-group | python3 -m json.tool

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync policy get
```

## Step 6: Test Selective Replication

```bash
# Upload to critical-data - should replicate
aws s3 cp test.txt s3://critical-data/ --endpoint-url http://rgw-us-east:80

# Upload to temp-data - should NOT replicate
aws s3 cp test.txt s3://temp-data/ --endpoint-url http://rgw-us-east:80

# Wait 30 seconds then check secondary zone
aws s3 ls s3://critical-data/ --endpoint-url http://rgw-us-west:80
aws s3 ls s3://temp-data/ --endpoint-url http://rgw-us-west:80
```

## Summary

RGW sync policies provide bucket-level and prefix-level control over multisite replication. Using group policies, you can selectively replicate business-critical data while excluding ephemeral or test buckets. Sync policy flows define which zone pairs participate in replication, enabling hub-and-spoke or selective peer models.
