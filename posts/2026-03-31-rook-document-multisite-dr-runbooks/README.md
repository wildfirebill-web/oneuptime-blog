# How to Document Multisite DR Runbooks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multisite, Runbook, DR, Documentation

Description: Create comprehensive disaster recovery runbooks for Ceph RGW multisite that enable any team member to execute failover and failback procedures reliably under pressure.

---

## Overview

DR runbooks are step-by-step procedures that enable any qualified engineer to execute a failover or failback without relying on tribal knowledge. A good runbook is specific, actionable, testable, and regularly updated. This guide provides templates and best practices for documenting Ceph RGW multisite DR procedures.

## Runbook Structure

Every DR runbook should include these sections:

```
1. Overview - what this runbook covers and when to use it
2. Prerequisites - tools, access, and knowledge required
3. Contact list - who to notify and escalate to
4. Pre-conditions - what to verify before starting
5. Procedure steps - numbered, specific, copy-pasteable commands
6. Verification steps - how to confirm success
7. Rollback steps - how to undo if the procedure fails
8. Estimated times - how long each step takes
```

## Failover Runbook Template

```markdown
# RGW Zone Failover Runbook

## Overview
Use this runbook when the primary RGW zone (us-east) is unavailable and
clients need to be redirected to the secondary zone (us-west).

## Estimated Time: 20-30 minutes

## Prerequisites
- SSH access to us-west RGW hosts
- Route 53 credentials for DNS update
- radosgw-admin installed on us-west hosts

## Step 1: Verify Primary is Unavailable (2 min)
Check if primary RGW is truly down (not just monitoring alerting):

    curl -s http://us-east-rgw.example.com:7480 | head -5
    # Expected: connection timeout or error

## Step 2: Check Secondary Sync Status (3 min)

    radosgw-admin sync status
    # Record: how many minutes behind the secondary was

## Step 3: Promote Secondary to Master (5 min)

    radosgw-admin zone modify --rgw-zone=us-west --master
    radosgw-admin period update --commit
    systemctl restart ceph-radosgw@rgw.us-west

## Step 4: Update DNS (5 min)

    # Change s3.example.com to point to us-west load balancer (10.0.2.100)
    aws route53 change-resource-record-sets --hosted-zone-id Z1234 \
        --change-batch file:///opt/runbooks/dns-failover.json

## Step 5: Verify (5 min)

    echo "failover-test" | aws s3 cp - s3://smoke-test/probe.txt \
        --endpoint-url http://us-west-rgw.example.com
    # Expected: upload success

## Rollback
If failover fails, restore DNS and demote us-west:
    # (Reverse DNS change)
    radosgw-admin zone modify --rgw-zone=us-east --master
    radosgw-admin period update --commit
```

## Storing and Versioning Runbooks

```bash
# Store runbooks in Git
git init /opt/runbooks
cd /opt/runbooks
git add .
git commit -m "Initial DR runbook for RGW multisite"

# Use tags for tested versions
git tag -a "tested-2026-01-15" -m "Validated in quarterly DR drill"
```

## DNS Change Templates

Pre-build your DNS change JSON files:

```json
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "s3.example.com",
      "Type": "A",
      "TTL": 60,
      "ResourceRecords": [{"Value": "10.0.2.100"}]
    }
  }]
}
```

```bash
# Save as /opt/runbooks/dns-failover.json
# And the reverse as /opt/runbooks/dns-failback.json
```

## Testing the Runbook

```bash
# Time yourself executing the runbook in a test environment
time bash /opt/runbooks/failover.sh

# Document the actual time taken
# Compare with the estimated time in the runbook
# Update if significantly different
```

## Runbook Review Schedule

```
- After every DR test or real incident: update with lessons learned
- Quarterly: review for accuracy, test commands still work
- Annually: full review including contact lists and DNS details
```

## Summary

Effective Ceph RGW DR runbooks are specific enough to execute under pressure, include pre-built copy-pasteable commands, and store pre-configured DNS change files. Version them in Git, tag validated versions after each drill, and update them immediately after real incidents. A runbook that has been tested and timed is far more valuable than a perfectly written but untested procedure.
