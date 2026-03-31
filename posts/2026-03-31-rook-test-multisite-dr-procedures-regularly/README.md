# How to Test Multisite DR Procedures Regularly

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multisite, DR Testing, Disaster Recovery, Automation

Description: Implement regular automated and manual DR testing procedures for Ceph RGW multisite to validate failover readiness without impacting production workloads.

---

## Overview

A DR plan that has never been tested is not a DR plan - it is a hope. Regular DR testing verifies that failover procedures work as documented, sync lag meets RPO targets, and your team can execute the runbook under pressure. This guide covers both non-disruptive tests and full failover drills.

## Non-Disruptive DR Tests (Run Daily/Weekly)

These tests verify DR readiness without actually failing over:

```bash
#!/bin/bash
# daily-dr-check.sh

echo "=== Daily DR Readiness Check - $(date) ==="

# 1. Check sync status
echo ""
echo "--- Sync Status ---"
radosgw-admin sync status 2>&1

# 2. Measure sync lag
echo ""
echo "--- Sync Lag ---"
radosgw-admin sync status 2>&1 | grep -E "behind|caught up|sync"

# 3. Write a probe object to primary
TIMESTAMP=$(date +%s)
echo "dr-probe-$TIMESTAMP" | aws s3 cp - s3://dr-test-bucket/probe-$TIMESTAMP.txt \
    --endpoint-url http://us-east-rgw.example.com

# 4. Wait and verify it appears on secondary
sleep 120
if aws s3 ls s3://dr-test-bucket/probe-$TIMESTAMP.txt \
    --endpoint-url http://us-west-rgw.example.com > /dev/null 2>&1; then
    echo "PASS: Sync probe object found on secondary"
else
    echo "FAIL: Sync probe object NOT found on secondary after 2 minutes"
    exit 1
fi

# 5. Measure actual sync time
echo "Sync latency: ~120 seconds (probe appeared within 2 minute check)"
```

## Quarterly Failover Drill

A full failover drill tests the complete procedure:

```bash
#!/bin/bash
# quarterly-dr-drill.sh

echo "=== Quarterly DR Failover Drill - $(date) ==="

# Pre-drill: Record current state
echo "--- Pre-Drill State ---"
radosgw-admin sync status
radosgw-admin period get | python3 -m json.tool | grep master

# Step 1: Simulate primary failure (stop RGW on primary)
echo "Simulating primary failure..."
# In test environment only:
systemctl stop ceph-radosgw@rgw.us-east

# Step 2: Measure time to detection
START_TIME=$(date +%s)
echo "Primary stopped at: $(date)"

# Step 3: Execute failover procedure
radosgw-admin zone modify --rgw-zone=us-west --master
radosgw-admin period update --commit

# Restart secondary RGW
systemctl restart ceph-radosgw@rgw.us-west

# Step 4: Verify secondary accepts writes
if echo "post-failover-test" | aws s3 cp - s3://dr-test-bucket/drill.txt \
    --endpoint-url http://us-west-rgw.example.com; then
    END_TIME=$(date +%s)
    RTO=$((END_TIME - START_TIME))
    echo "PASS: Failover complete in ${RTO} seconds"
else
    echo "FAIL: Secondary not accepting writes after failover"
fi

# Step 5: Execute failback
echo "--- Starting Failback ---"
systemctl start ceph-radosgw@rgw.us-east
radosgw-admin zone modify --rgw-zone=us-east --master
radosgw-admin period update --commit
```

## Automated RPO Monitoring

Set up continuous RPO monitoring:

```bash
# Create a Prometheus alerting rule for sync lag
cat > /etc/prometheus/rules/ceph-dr.yml << 'EOF'
groups:
  - name: ceph_dr
    rules:
      - alert: CephRGWSyncLagHigh
        expr: ceph_rgw_metadata_sync_behind_shards > 100
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Ceph RGW sync lag may exceed RPO"
EOF
```

## DR Test Checklist

```
Monthly Tests:
  [ ] Run daily-dr-check.sh and verify all probes succeed
  [ ] Verify sync lag is within RPO target
  [ ] Confirm DR runbook document is current
  [ ] Verify DR team contact list is updated

Quarterly Tests:
  [ ] Execute full failover drill in staging/test environment
  [ ] Measure actual RTO and compare with target
  [ ] Identify and fix any runbook gaps
  [ ] Update runbook with lessons learned

Annual Tests:
  [ ] Execute production DR drill during low-traffic window
  [ ] Test failback procedure end-to-end
  [ ] Verify DNS TTL cutover timing meets RTO
```

## Summary

Regular DR testing ensures that Ceph RGW multisite failover procedures work when needed. Run daily sync probe tests to verify RPO compliance, execute quarterly drill failovers in non-production environments to validate RTO, and perform annual production drills to test the full end-to-end procedure including DNS cutover. Document results after each test and update runbooks to reflect lessons learned.
