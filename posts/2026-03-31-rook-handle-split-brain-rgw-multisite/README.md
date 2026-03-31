# How to Handle Split-Brain in Ceph RGW Multisite

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multisite, Split-Brain, Consistency, Disaster Recovery

Description: Detect and resolve split-brain scenarios in Ceph RGW multisite where two zones simultaneously accept writes, causing metadata conflicts and data divergence.

---

## Overview

A split-brain in Ceph RGW multisite occurs when both zones simultaneously accept writes because each believes it is the master zone. This happens when the network between zones is partitioned and the secondary was promoted to master for DR. When connectivity is restored, both zones have divergent data that must be reconciled.

## Detecting Split-Brain

```bash
# Check if both zones claim to be master
# On zone A
radosgw-admin zone get --rgw-zone=us-east | grep '"is_master"'

# On zone B
radosgw-admin zone get --rgw-zone=us-west | grep '"is_master"'

# If both return "is_master": true, you have a split-brain
```

## Understanding the Impact

During split-brain, the same object key may be written in both zones with different content. RGW uses timestamps and version IDs to determine which version wins during sync.

```bash
# Check for conflicting objects
radosgw-admin log list --log-type=data

# View pending sync operations on each zone
radosgw-admin data sync status --source-zone=us-east
radosgw-admin data sync status --source-zone=us-west
```

## Resolution Strategy

The recommended approach is to designate one zone as authoritative and resync the other:

```bash
# Step 1: Stop writes to the zone you will discard
# Put the secondary (us-west) into read-only or stop RGW
systemctl stop ceph-radosgw@rgw.us-west

# Step 2: Identify conflicts - objects written to both zones
radosgw-admin object stat --bucket=mybucket --object=contested-object \
    --rgw-zone=us-east

radosgw-admin object stat --bucket=mybucket --object=contested-object \
    --rgw-zone=us-west
```

## Resolving Metadata Conflicts

```bash
# List metadata that differs between zones
radosgw-admin metadata list bucket

# Sync metadata from authoritative zone to secondary
radosgw-admin metadata sync run --source-zone=us-east

# Force re-sync a specific bucket from authoritative zone
radosgw-admin bucket sync run --bucket=mybucket --source-zone=us-east
```

## Data Reconciliation

For critical buckets where you cannot afford to lose either side's writes:

```bash
# Export object list from both zones
aws s3 ls s3://mybucket --recursive --endpoint-url http://us-east-rgw.example.com \
    > /tmp/us-east-objects.txt

aws s3 ls s3://mybucket --recursive --endpoint-url http://us-west-rgw.example.com \
    > /tmp/us-west-objects.txt

# Find objects only in secondary (written during split-brain)
diff /tmp/us-east-objects.txt /tmp/us-west-objects.txt | grep "^>" > /tmp/split-brain-objects.txt

# Manually copy unique objects from secondary to a safe location
while read line; do
    OBJ=$(echo $line | awk '{print $4}')
    aws s3 cp s3://mybucket/$OBJ s3://recovery-bucket/$OBJ \
        --endpoint-url http://us-west-rgw.example.com
done < /tmp/split-brain-objects.txt
```

## Preventing Future Split-Brain

```bash
# Use a witness or arbiter zone to prevent dual-master promotion
# Set a minimum number of zones required to accept writes
radosgw-admin zone modify --rgw-zone=us-east --tier-config=write-quorum=2

# Configure a shorter sync period to reduce split-brain window
ceph config set client.rgw sync_from_all true
ceph config set client.rgw rgw_zone_sync_period_hours 1
```

## Post-Resolution Verification

```bash
# Verify only one zone is master
radosgw-admin period get | python3 -m json.tool | grep -A5 "master"

# Verify sync is healthy
radosgw-admin sync status

# Run data integrity check
radosgw-admin data sync status
```

## Summary

Split-brain in Ceph RGW multisite occurs when network partitions allow both zones to accept writes simultaneously. Resolution requires designating one zone as authoritative, extracting unique objects from the discarded zone, and resyncing metadata. Prevent future split-brain by using short DNS TTLs, coordinated failover procedures, and sync health monitoring to detect divergence early.
