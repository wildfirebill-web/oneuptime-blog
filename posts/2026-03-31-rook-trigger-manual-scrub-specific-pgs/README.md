# How to Trigger Manual Scrub on Specific PGs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Scrubbing, PG, Data Integrity, Kubernetes, Maintenance

Description: Learn how to trigger manual shallow and deep scrubs on specific placement groups in Ceph to verify data integrity on demand without waiting for scheduled scrubs.

---

## When to Trigger Manual Scrubs

Manual scrubs are useful in several scenarios:
- After recovering from a disk failure or OSD replacement
- When investigating suspected data corruption
- Before decommissioning hardware
- After restoring data from backup
- When a PG has not been scrubbed within the expected interval

## Listing Placement Groups

Before triggering scrubs, identify the PGs you want to target:

```bash
# List all PGs with their current state
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg dump | grep -v "^$" | head -30

# List PGs in a specific pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg ls-by-pool my-pool

# Find PGs on a specific OSD
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg ls-by-osd osd.3
```

PG IDs have the format `<pool-id>.<pg-number>` (e.g., `1.2f`, `3.1a`).

## Triggering a Shallow Scrub

Initiate a shallow scrub on a specific PG:

```bash
# Scrub a single PG
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg scrub 1.2f

# Scrub all PGs in a pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool scrub my-pool
```

## Triggering a Deep Scrub

Initiate a full data integrity check:

```bash
# Deep scrub a single PG
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg deep-scrub 1.2f

# Deep scrub all PGs in a pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool deep-scrub my-pool
```

## Scrubbing All PGs on a Specific OSD

When replacing or repairing an OSD, scrub all its PGs:

```bash
# Scrub all PGs on OSD 3
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c \
  "for pg in \$(ceph pg ls-by-osd osd.3 | awk 'NR>1 {print \$1}'); do
    echo \"Scrubbing PG \$pg\"
    ceph pg scrub \$pg
  done"
```

## Monitoring Scrub Progress

Watch scrub progress in real time:

```bash
# Check which PGs are currently being scrubbed
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 2 "ceph pg dump | grep scrubbing"

# Check the status of a specific PG
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg 1.2f query | python3 -m json.tool | grep -E "state|last_scrub|last_deep"
```

## Checking Scrub Results

After a scrub completes, check for inconsistencies:

```bash
# Check health for scrub-related issues
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail | grep -E "inconsistent|scrub"

# Check specific PG for inconsistencies
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg 1.2f query | python3 -m json.tool | grep "inconsistent"
```

## Scrubbing After OSD Replacement

After adding a replacement OSD, trigger immediate scrubs on its PGs:

```bash
# Wait for PGs to become active
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg stat

# Trigger deep scrub on all affected PGs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c \
  "ceph pg ls-by-osd osd.5 2>/dev/null | awk 'NR>1 {print \$1}' | \
  xargs -I{} ceph pg deep-scrub {}"
```

## Summary

Manual scrub triggering gives you on-demand data integrity verification for specific placement groups or pools. Use shallow scrubs (`ceph pg scrub`) for quick metadata consistency checks and deep scrubs (`ceph pg deep-scrub`) for full data integrity verification. Always trigger manual scrubs after hardware replacement or recovery operations, and monitor results through `ceph health detail` to catch any inconsistencies that need repair.
