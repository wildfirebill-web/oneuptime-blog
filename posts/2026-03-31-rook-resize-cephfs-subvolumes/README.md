# How to Resize CephFS Subvolumes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Resize, Subvolume, Quota, Storage

Description: Expand or shrink CephFS subvolume quotas dynamically without downtime, allowing you to adjust storage allocations as tenant needs change.

---

CephFS subvolume sizing is controlled by quota attributes. Unlike block devices, CephFS volumes do not have a fixed size - the "size" is a quota that can be changed at any time without unmounting or interrupting the tenant. This guide covers expanding, shrinking, and removing quotas.

## How Resizing Works

Resizing a CephFS subvolume changes the quota extended attribute on the subvolume's root directory. The change takes effect immediately - the tenant's writes will be blocked at the new limit as soon as the MDS enforces the updated quota. No reformatting, unmounting, or restart is needed.

## Checking Current Size and Usage

Before resizing, check the current state:

```bash
# Get current quota and usage
ceph fs subvolume info cephfs webapp --format json | \
  python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f\"Quota: {d['bytes_quota'] // 1073741824} GB\")
print(f\"Used:  {d['bytes_used'] // 1073741824} GB\")
print(f\"Pct:   {d['bytes_pcent']}%\")
"
```

## Expanding a Subvolume

```bash
# Expand webapp from 10 GB to 50 GB
ceph fs subvolume resize cephfs webapp 53687091200

# Expand a subvolume in a group
ceph fs subvolume resize cephfs app1 \
  21474836480 \
  --group_name production

# Verify new quota
ceph fs subvolume info cephfs webapp | grep bytes_quota
```

## Shrinking a Subvolume

```bash
# Attempt to shrink to 5 GB
ceph fs subvolume resize cephfs webapp 5368709120
# If current usage > 5 GB, this returns an error:
# Error EINVAL: requested size 5368709120 is less than current used size 8589934592
```

To shrink below current usage, you must first reduce usage:

```bash
# Check what is using space
du -sh /mnt/webapp/*

# After removing or moving data, resize
ceph fs subvolume resize cephfs webapp 5368709120
```

To force the shrink (new writes will immediately be blocked):

```bash
ceph fs subvolume resize cephfs webapp 5368709120 --no_shrink_check
```

## Removing the Quota (Unlimited)

```bash
# Set size to 0 to remove the quota
ceph fs subvolume resize cephfs webapp 0

# Verify quota is gone
ceph fs subvolume info cephfs webapp | grep bytes_quota
# "bytes_quota": "infinite"
```

## Bulk Resize Script

```bash
#!/bin/bash
# bulk-resize.sh - Resize all subvolumes in a group by a percentage

FS="cephfs"
GROUP="production"
FACTOR=2   # double all quotas

for sv in $(ceph fs subvolume ls ${FS} --group_name ${GROUP} \
  --format json | python3 -c "import sys,json; [print(s['name']) for s in json.load(sys.stdin)]"); do

  CURRENT=$(ceph fs subvolume info ${FS} ${sv} \
    --group_name ${GROUP} --format json | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('bytes_quota',0))")

  if [ "${CURRENT}" != "0" ] && [ -n "${CURRENT}" ]; then
    NEW=$(( CURRENT * FACTOR ))
    echo "Resizing ${sv}: ${CURRENT} -> ${NEW}"
    ceph fs subvolume resize ${FS} ${sv} ${NEW} --group_name ${GROUP}
  fi
done
```

## Summary

CephFS subvolume resizing is a live operation that adjusts quota attributes without any service interruption. Expanding is always safe; shrinking requires current usage to be below the new limit unless forced with `--no_shrink_check`. Setting size to 0 removes the quota entirely. These properties make CephFS ideal for dynamic multi-tenant storage where allocations need to grow and shrink as application requirements change, without the complexity of block device resize operations.
