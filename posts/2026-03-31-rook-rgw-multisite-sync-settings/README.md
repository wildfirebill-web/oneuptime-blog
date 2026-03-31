# How to Set Multisite Sync Settings in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multisite, Zone, Replication

Description: Configure rgw_zone, rgw_zonegroup, and rgw_realm settings in Ceph RGW to set up multisite object replication across data centers or Kubernetes clusters.

---

Ceph RGW multisite allows you to replicate object data across multiple zones in different data centers. Three key parameters - `rgw_zone`, `rgw_zonegroup`, and `rgw_realm` - define the placement of each RGW instance in the multisite hierarchy.

## Multisite Hierarchy

```
Realm
  Zone Group (zonegroup)
    Zone
      RGW Instance
```

- **Realm** - Top-level namespace, unique across all sites
- **Zonegroup** - Group of zones that share metadata
- **Zone** - Individual storage site with its own data pools

## Setting Up a Primary Zone

```bash
# Create realm
radosgw-admin realm create --rgw-realm=production --default

# Create zonegroup
radosgw-admin zonegroup create --rgw-zonegroup=us --endpoints=http://rgw1.example.com:80 --master --default

# Create master zone
radosgw-admin zone create --rgw-zonegroup=us --rgw-zone=us-east-1 \
  --access-key=SYSTEM_ACCESS_KEY --secret=SYSTEM_SECRET_KEY --master --default

# Create system user for replication
radosgw-admin user create --uid=zone-user --display-name="Zone User" \
  --access-key=SYSTEM_ACCESS_KEY --secret=SYSTEM_SECRET_KEY --system
```

## Setting Zone Parameters in ceph.conf

```bash
# Apply zone settings via config database
ceph config set client.rgw rgw_zone us-east-1
ceph config set client.rgw rgw_zonegroup us
ceph config set client.rgw rgw_realm production
```

## Setting Up a Secondary Zone

On the secondary cluster:

```bash
# Pull realm from primary
radosgw-admin realm pull --url=http://rgw1.example.com:80 \
  --access-key=SYSTEM_ACCESS_KEY --secret=SYSTEM_SECRET_KEY

# Create the secondary zone
radosgw-admin zone create --rgw-zonegroup=us --rgw-zone=us-west-1 \
  --access-key=SYSTEM_ACCESS_KEY --secret=SYSTEM_SECRET_KEY

# Update period
radosgw-admin period update --commit
```

## Applying in Rook with CephObjectZone

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZone
metadata:
  name: us-east-1
  namespace: rook-ceph
spec:
  zoneGroup: us
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
---
apiVersion: ceph.rook.io/v1
kind: CephObjectZoneGroup
metadata:
  name: us
  namespace: rook-ceph
spec:
  realm: production
---
apiVersion: ceph.rook.io/v1
kind: CephObjectRealm
metadata:
  name: production
  namespace: rook-ceph
```

## Verifying Multisite Configuration

```bash
# Verify zone config
radosgw-admin zone get --rgw-zone=us-east-1

# Check sync status
radosgw-admin sync status

# View replication lag
radosgw-admin data sync status --source-zone=us-east-1
```

## Summary

Ceph RGW multisite replication is configured via `rgw_zone`, `rgw_zonegroup`, and `rgw_realm` parameters, plus the realm/zonegroup/zone objects. In Rook, use `CephObjectRealm`, `CephObjectZoneGroup`, and `CephObjectZone` CRDs to manage the hierarchy declaratively. Always run `radosgw-admin sync status` to verify replication health.
