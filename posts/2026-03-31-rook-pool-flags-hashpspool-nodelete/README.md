# How to Set Pool Flags in Ceph (HASHPSPOOL, NODELETE, NOPGCHANGE, NOSIZECHANGE)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool, Flag, Configuration

Description: Configure Ceph pool protection flags including HASHPSPOOL, NODELETE, NOPGCHANGE, and NOSIZECHANGE to prevent accidental operational changes.

---

Ceph pools support several boolean flags that control both safety behavior and internal operations. These flags act as guardrails preventing accidental changes to pool configuration in production environments.

## Pool Flags Overview

| Flag | Purpose | Default |
|---|---|---|
| `hashpspool` | Improves hash distribution across PGs | true (on new pools) |
| `nodelete` | Prevents pool deletion | false |
| `nopgchange` | Prevents PG count changes | false |
| `nosizechange` | Prevents replica size changes | false |
| `write_fadvise_dontneed` | Hints OS to not cache writes | false |

## Access Rook Toolbox

```bash
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- bash
```

## HASHPSPOOL Flag

`hashpspool` enables a more uniform hash distribution of objects across placement groups. It is recommended for all pools and is enabled by default on pools created in Ceph Luminous and later.

```bash
# Enable (recommended)
ceph osd pool set replicapool hashpspool true

# Check current state
ceph osd pool get replicapool hashpspool
```

Avoid disabling this flag unless you are rolling back compatibility with a very old Ceph version.

## NODELETE Flag

The `nodelete` flag prevents a pool from being deleted. Useful for protecting critical production pools:

```bash
# Protect a pool from deletion
ceph osd pool set replicapool nodelete true

# Verify protection
ceph osd pool get replicapool nodelete

# Remove protection before intentional deletion
ceph osd pool set replicapool nodelete false
```

When `nodelete` is set to true, `ceph osd pool delete` will return an error even with the `--yes-i-really-really-mean-it` flag.

## NOPGCHANGE Flag

The `nopgchange` flag prevents modification of the `pg_num` and `pgp_num` values for a pool. Useful when you have carefully tuned PG counts and want to prevent accidental changes:

```bash
# Lock PG count
ceph osd pool set replicapool nopgchange true

# Verify
ceph osd pool get replicapool nopgchange

# Unlock for intentional PG changes
ceph osd pool set replicapool nopgchange false
```

## NOSIZECHANGE Flag

The `nosizechange` flag prevents modification of the `size` and `min_size` replica parameters:

```bash
# Prevent replica count changes
ceph osd pool set replicapool nosizechange true

# Verify
ceph osd pool get replicapool nosizechange

# Unlock for intentional size changes
ceph osd pool set replicapool nosizechange false
```

## Check All Flags at Once

```bash
ceph osd pool get replicapool all
```

Or use JSON output to parse flags programmatically:

```bash
ceph osd dump --format json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for pool in data['pools']:
    if pool['pool_name'] == 'replicapool':
        flags = pool.get('flags_names', '')
        print(f\"Flags: {flags}\")
"
```

## Apply Flags via CephBlockPool CRD Parameters

In Rook, you can set these flags through the `parameters` section of the CRD, though not all flags are exposed. For safety flags, prefer CLI management as they are operational controls rather than provisioning configuration:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    hashpspool: "true"
```

## Summary

Ceph pool flags like `nodelete`, `nopgchange`, and `nosizechange` act as operational locks that prevent accidental modification of critical pool properties. Enable `nodelete` on production pools to guard against accidental data loss, and set `nopgchange` and `nosizechange` after initial configuration is stable. Always verify and clear these flags before performing intentional maintenance operations.
