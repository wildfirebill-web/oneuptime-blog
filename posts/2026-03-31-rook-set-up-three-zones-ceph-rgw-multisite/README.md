# How to Set Up Three or More Zones in Ceph RGW Multisite

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Multisite, Zone, Replication, Storage, Kubernetes

Description: Step-by-step guide to configuring three or more zones in a Ceph RGW multisite deployment for geo-distributed object storage.

---

## Overview

While a two-zone Ceph RGW multisite setup provides basic redundancy, adding a third or more zones enables multi-region data availability and can reduce read latency for geographically distributed users. This guide walks through adding a third zone to an existing two-zone multisite deployment.

## Existing Topology Assumption

This guide assumes you have:
- Zone1 (master, us-east-1)
- Zone2 (secondary, us-west-1)
- Zone3 (new, eu-central-1) - to be added

## Step 1 - Create the Third Zone

On the master zone, create the new zone definition:

```bash
radosgw-admin zone create \
  --rgw-zonegroup=global \
  --rgw-zone=zone3 \
  --access-key=ZONE3_ACCESS_KEY \
  --secret=ZONE3_SECRET_KEY \
  --endpoints=http://zone3-rgw.example.com
```

## Step 2 - Create a System User for Zone3

```bash
radosgw-admin user create \
  --uid=zone3.user \
  --display-name="Zone3 Sync User" \
  --access-key=ZONE3_ACCESS_KEY \
  --secret=ZONE3_SECRET_KEY \
  --system
```

## Step 3 - Update the Zone Group and Commit

Add zone3 to the zone group and commit the period:

```bash
radosgw-admin zonegroup add \
  --rgw-zonegroup=global \
  --rgw-zone=zone3

radosgw-admin period update --commit
```

Verify the zone group now shows three zones:

```bash
radosgw-admin zonegroup get | python3 -c "
import sys, json
zg = json.load(sys.stdin)
for z in zg['zones']:
    print('Zone:', z['name'], '| Endpoints:', z['endpoints'])
"
```

## Step 4 - Configure Zone3 Cluster

On the zone3 cluster, pull the realm and period from the master:

```bash
radosgw-admin realm pull \
  --url=http://zone1-rgw.example.com \
  --access-key=ZONE3_ACCESS_KEY \
  --secret=ZONE3_SECRET_KEY

radosgw-admin period pull \
  --url=http://zone1-rgw.example.com \
  --access-key=ZONE3_ACCESS_KEY \
  --secret=ZONE3_SECRET_KEY
```

Configure zone3's local settings:

```bash
radosgw-admin zone modify \
  --rgw-zone=zone3 \
  --access-key=ZONE3_ACCESS_KEY \
  --secret=ZONE3_SECRET_KEY

radosgw-admin period update --commit
```

## Step 5 - Ceph Configuration for Zone3

Add the zone group and zone settings to zone3's `ceph.conf`:

```ini
[client.rgw.zone3]
rgw_zone = zone3
rgw_zonegroup = global
rgw_realm = myrealm
rgw_frontends = beast endpoint=0.0.0.0:80
```

Start the RGW service:

```bash
systemctl enable --now ceph-radosgw@rgw.zone3
```

## Step 6 - Verify Three-Zone Sync

Check that all three zones are syncing:

```bash
radosgw-admin sync status
```

Expected output includes data sync entries for both zone1 and zone2 as source zones. Write a test object and verify it appears on all zones:

```bash
aws s3 cp /tmp/test.txt s3://test-bucket/ --endpoint-url http://zone1-rgw.example.com
aws s3 ls s3://test-bucket/ --endpoint-url http://zone2-rgw.example.com
aws s3 ls s3://test-bucket/ --endpoint-url http://zone3-rgw.example.com
```

## Summary

Adding a third zone to Ceph RGW multisite requires creating the zone definition on the master, adding a system user for sync credentials, committing the updated period, and then pulling the realm and period configuration on the new zone cluster. Each additional zone participates in full bidirectional data sync, providing geo-redundancy and read locality for distributed workloads.
