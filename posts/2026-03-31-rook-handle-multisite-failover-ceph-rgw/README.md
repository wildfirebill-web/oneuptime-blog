# How to Handle Multisite Failover in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Multisite, Failover, Disaster Recovery, Storage, Kubernetes

Description: Learn how to perform controlled and emergency failover in Ceph RGW multisite deployments to maintain availability when a zone goes offline.

---

## Overview

Ceph RGW multisite replication allows you to distribute object storage across multiple zones. When a primary zone becomes unavailable, you need to promote a secondary zone to become the new primary. This guide covers both graceful failover and emergency failover procedures.

## Prerequisites

- A working Ceph RGW multisite deployment with at least two zones
- Access to `radosgw-admin` on both sites
- Understanding of your zone group and realm topology

Check your current realm and zone group configuration:

```bash
radosgw-admin realm list
radosgw-admin zonegroup list
radosgw-admin zone list
```

## Understanding Failover Types

There are two types of failover in Ceph RGW multisite:

- **Graceful failover** - Primary zone is still accessible; you intentionally promote secondary
- **Emergency failover** - Primary zone is down; you must force-promote secondary

## Graceful Failover

When the primary zone is still reachable, first verify sync is complete:

```bash
radosgw-admin sync status
```

On the secondary zone, promote it to master:

```bash
radosgw-admin zone modify --rgw-zone=zone2 --master
radosgw-admin zonegroup modify --rgw-zonegroup=us-east --master-zone=zone2
radosgw-admin period update --commit
```

Restart RGW instances on the newly promoted zone:

```bash
systemctl restart ceph-radosgw@*
```

## Emergency Failover

When the primary zone is unreachable, use the `--yes-i-really-mean-it` flag:

```bash
radosgw-admin zone modify --rgw-zone=zone2 --master --yes-i-really-mean-it
radosgw-admin zonegroup modify --rgw-zonegroup=us-east --master-zone=zone2 --yes-i-really-mean-it
radosgw-admin period update --commit --yes-i-really-mean-it
```

Update DNS to point clients to the new zone endpoint:

```bash
# Update Route53 or your DNS provider to point to zone2 endpoint
# Example using AWS CLI:
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567 \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"s3.example.com","Type":"A","TTL":60,"ResourceRecords":[{"Value":"10.0.2.100"}]}}]}'
```

## Verifying Failover Success

After promoting the secondary zone, verify it is now the master:

```bash
radosgw-admin period get | python3 -c "
import sys, json
period = json.load(sys.stdin)
for zg in period['period_map']['zonegroups']:
    print('Master zone:', zg['master_zone'])
"
```

Check that data is accessible:

```bash
aws s3 ls s3://my-bucket --endpoint-url http://zone2-rgw.example.com
```

## Recovering the Former Primary

Once the original primary zone is restored, re-add it as a secondary:

```bash
# On restored zone1, pull the new period
radosgw-admin period pull --url=http://zone2-rgw.example.com --access-key=KEY --secret=SECRET

# Let zone1 sync from zone2
radosgw-admin zone modify --rgw-zone=zone1 --access-key=KEY --secret=SECRET
radosgw-admin period update --commit
```

## Summary

Handling multisite failover in Ceph RGW involves promoting a secondary zone to master using `radosgw-admin` commands, with graceful failover requiring clean sync status verification and emergency failover using the `--yes-i-really-mean-it` flag. After failover, clients must be redirected to the new zone endpoint, and the recovered zone can later be re-added as a secondary for ongoing replication.
