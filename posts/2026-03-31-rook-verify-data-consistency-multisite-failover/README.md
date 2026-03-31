# How to Verify Data Consistency After Multisite Failover

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multisite, Data Consistency, Failover, Verification

Description: Verify data consistency in Ceph RGW after a multisite failover by comparing object counts, checking sync status, and running bucket integrity checks on the secondary zone.

---

## Overview

After failing over to the secondary Ceph RGW zone, it is critical to verify data consistency before trusting the secondary for production workloads. This involves checking object counts, verifying bucket metadata, and confirming no corruption occurred during the transition.

## Step 1 - Check Sync Status at Failover Time

```bash
# View sync status on the secondary (now master) zone
radosgw-admin sync status

# Check how far behind the secondary was at failover time
radosgw-admin sync status | grep -E "behind|caught|sync"

# View sync log entries
radosgw-admin log list --log-type=data | head -20
```

## Step 2 - Compare Object Counts

Compare bucket object counts between the zones:

```bash
# On secondary (current master) zone
radosgw-admin bucket stats --bucket=critical-bucket | python3 -c "
import sys, json
data = json.load(sys.stdin)
usage = data.get('usage', {}).get('rgw.main', {})
print('Objects:', usage.get('num_objects', 0))
print('Bytes:', usage.get('size_actual', 0))
"
```

```bash
# On primary zone (if accessible) - compare counts
radosgw-admin bucket stats --bucket=critical-bucket \
    --rgw-zone=us-east | python3 -c "
import sys, json
data = json.load(sys.stdin)
usage = data.get('usage', {}).get('rgw.main', {})
print('Objects:', usage.get('num_objects', 0))
print('Bytes:', usage.get('size_actual', 0))
"
```

## Step 3 - Run Bucket Integrity Check

```bash
# Run a bucket check to verify object metadata consistency
radosgw-admin bucket check --bucket=critical-bucket

# Check for orphaned objects
radosgw-admin bucket check --bucket=critical-bucket --check-objects

# Fix any inconsistencies found
radosgw-admin bucket check --bucket=critical-bucket --fix
```

## Step 4 - Verify Object Readability

```bash
#!/bin/bash
# verify-objects.sh - test that critical objects are readable

BUCKET="critical-bucket"
ENDPOINT="http://us-west-rgw.example.com"
ERRORS=0

# Get list of objects to verify
aws s3 ls s3://$BUCKET --endpoint-url $ENDPOINT | awk '{print $4}' | \
while read obj; do
    if aws s3 cp s3://$BUCKET/$obj /dev/null --endpoint-url $ENDPOINT 2>/dev/null; then
        echo "OK: $obj"
    else
        echo "ERROR: $obj"
        ((ERRORS++))
    fi
done

echo "Errors: $ERRORS"
```

## Step 5 - Check for Incomplete Multipart Uploads

```bash
# List incomplete multipart uploads (potential in-flight writes at failover time)
radosgw-admin incomplete-multipart list --bucket=critical-bucket

# Remove stale incomplete multiparts older than 24 hours
radosgw-admin incomplete-multipart rm --bucket=critical-bucket \
    --num-shards=16 --remove-delay=86400
```

## Step 6 - Validate Bucket Versioning State

```bash
# Check versioning state is consistent
radosgw-admin bucket versioning --bucket=critical-bucket

# Verify versioned objects have correct version IDs
aws s3api list-object-versions --bucket critical-bucket \
    --endpoint-url http://us-west-rgw.example.com | \
    python3 -m json.tool | grep '"VersionId"' | wc -l
```

## Generating a Consistency Report

```bash
#!/bin/bash
# consistency-report.sh

echo "=== Ceph RGW Post-Failover Consistency Report ==="
echo "Date: $(date)"
echo "Secondary zone: us-west"
echo ""

echo "=== Sync Status ==="
radosgw-admin sync status 2>&1 | head -20

echo ""
echo "=== Bucket Summary ==="
for bucket in $(radosgw-admin bucket list 2>/dev/null); do
    COUNT=$(radosgw-admin bucket stats --bucket=$bucket 2>/dev/null | \
        python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('usage',{}).get('rgw.main',{}).get('num_objects',0))" 2>/dev/null)
    echo "  $bucket: $COUNT objects"
done
```

## Summary

Post-failover data consistency verification in Ceph RGW involves checking sync lag at the time of failover (to understand the RPO), comparing object counts between zones, running `radosgw-admin bucket check` for metadata integrity, testing object readability, and cleaning up stale incomplete multipart uploads. Document the results as the post-incident record for your RPO compliance reporting.
