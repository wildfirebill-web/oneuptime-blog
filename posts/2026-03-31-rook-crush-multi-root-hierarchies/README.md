# How to Set Up Multi-Root CRUSH Hierarchies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CRUSH, Storage, Configuration

Description: Learn how to configure multiple CRUSH root buckets in Ceph to support isolated storage domains, different hardware generations, or separate tenant namespaces.

---

## What are Multi-Root CRUSH Hierarchies

A multi-root CRUSH hierarchy contains more than one root bucket, each representing an independent storage domain. Different pools can be assigned to different roots, ensuring complete isolation between their data placement. This is useful for:

- Separating development/production storage environments
- Isolating different hardware generations (old HDDs vs new NVMe)
- Supporting multi-tenant deployments where tenants must have dedicated hardware
- Creating geographically separate storage zones

## Creating Multiple Root Buckets

```bash
# View the current hierarchy (one root by default)
ceph osd tree

# Create a second root bucket
ceph osd crush add-bucket secondary-root root

# Create a third root for an isolated tier
ceph osd crush add-bucket nvme-root root

# Verify all roots exist
ceph osd crush dump | python3 -m json.tool | grep '"type_name": "root"' -B3
```

## Building the Hierarchy Under Each Root

```bash
# Create and place hosts under the secondary root
ceph osd crush add-bucket node-backup-01 host
ceph osd crush add-bucket node-backup-02 host
ceph osd crush move node-backup-01 root=secondary-root
ceph osd crush move node-backup-02 root=secondary-root

# Assign OSDs to the backup hosts
ceph osd crush set osd.6 2.0 root=secondary-root host=node-backup-01
ceph osd crush set osd.7 2.0 root=secondary-root host=node-backup-01
ceph osd crush set osd.8 2.0 root=secondary-root host=node-backup-02
ceph osd crush set osd.9 2.0 root=secondary-root host=node-backup-02

# Verify the secondary hierarchy
ceph osd tree
```

## Creating CRUSH Rules for Each Root

Each pool targeting a specific root needs a rule that starts from that root:

```bash
# Rule for the primary (default) root
ceph osd crush rule create-replicated primary-rule default host

# Rule for the secondary root
ceph osd crush rule create-replicated backup-rule secondary-root host

# Rule for the NVMe root
ceph osd crush rule create-replicated nvme-rule nvme-root host

# List all rules
ceph osd crush rule ls
```

## Assigning Pools to Roots via Rules

```bash
# Create a production pool using the primary hierarchy
ceph osd pool create production 128 128 replicated primary-rule

# Create a backup pool using the secondary hierarchy
ceph osd pool create backups 64 64 replicated backup-rule

# Verify pool isolation
ceph osd map production testobj1    # should show OSDs 0-5
ceph osd map backups testobj2       # should show OSDs 6-9
```

## Multi-Root in Rook

In Rook, multi-root isolation is achieved with separate device sets pointing to distinct nodes or storage classes:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    storageClassDeviceSets:
      - name: primary-set
        count: 3
        placement:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
                - matchExpressions:
                    - key: storage-tier
                      operator: In
                      values: ["primary"]
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              storageClassName: fast-storage
              resources:
                requests:
                  storage: 1Ti
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

## Verifying Root Isolation

```bash
# Confirm the OSD tree shows separate root hierarchies
ceph osd tree

# Check that PGs for each pool use only OSDs in their respective root
ceph pg dump pgs | awk '{print $1, $14}' | \
  while read pg osds; do
    pool=$(echo $pg | cut -d. -f1)
    echo "PG $pg -> OSDs $osds"
  done | head -20
```

## Summary

Multi-root CRUSH hierarchies provide complete isolation between storage pools by assigning them to separate root buckets that share no OSDs. Create additional roots with `ceph osd crush add-bucket`, populate them with hosts and OSDs, create CRUSH rules that `take` from those specific roots, and assign those rules to the pools that need isolation. This enables true multi-tenant or multi-tier storage within a single Ceph deployment.
