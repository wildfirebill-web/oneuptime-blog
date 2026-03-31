# How to Assign MDS Daemons to Ranks in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS

Description: Learn how MDS ranks work in CephFS, how to assign MDS daemons to specific ranks, and how to scale active MDS counts in Rook deployments.

---

## What Are MDS Ranks

In CephFS, a "rank" is an active MDS slot that serves a portion of the filesystem's namespace. By default, CephFS uses a single rank (rank 0) for the entire namespace. You can scale to multiple active MDS daemons by increasing the `max_mds` setting, which creates additional ranks.

Each rank is served by exactly one active MDS daemon. Additional MDS daemons that are not assigned to a rank become standby daemons, ready to take over if an active MDS fails.

## Checking Current MDS State

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

```bash
ceph mds stat
```

Sample output:

```text
myfs:2 {0=myfs-a=up:active,1=myfs-b=up:active} 1 up:standby
```

- Two ranks (0 and 1) are active, served by `myfs-a` and `myfs-b`
- One standby MDS is available

## Getting Detailed MDS Information

```bash
ceph fs status myfs
```

Sample output:

```text
myfs - 4 clients
====
RANK  STATE      MDS             ACTIVITY     DNS    INOS
 0    active     myfs-a(up)      Reqs:    5   220    215
 1    active     myfs-b(up)      Reqs:    2   80     78

STANDBY MDS
myfs-c
```

## Increasing Active MDS Count (Scaling Up)

Increase the number of active ranks using `max_mds`:

```bash
ceph fs set myfs max_mds 2
```

This creates a second active rank. A standby MDS daemon will be promoted to fill it:

```bash
ceph mds stat
# Shows 2 active MDS daemons serving ranks 0 and 1
```

## Decreasing Active MDS Count (Scaling Down)

Decrease to a single active MDS:

```bash
ceph fs set myfs max_mds 1
```

Rank 1 is gracefully failed back and its metadata is migrated to rank 0. The MDS serving rank 1 becomes a standby.

## Managing MDS Ranks in Rook

In Rook, the number of active MDS daemons is controlled by `activeCount` in the `CephFilesystem` CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataServer:
    activeCount: 2         # Number of active ranks
    activeStandby: true    # Each active rank has a warm standby
```

Apply the change:

```bash
kubectl apply -f cephfilesystem.yaml
```

Rook deploys the necessary MDS pods and sets `max_mds` accordingly.

## Pinning Directories to Ranks

With multiple active MDS ranks, you can pin subtrees to specific ranks for predictable locality:

```bash
# Pin /data directory to rank 1
setfattr -n ceph.dir.pin -v 1 /mnt/myfs/data

# Pin /logs directory to rank 0
setfattr -n ceph.dir.pin -v 0 /mnt/myfs/logs
```

This improves performance by co-locating related files on the same rank, reducing cross-rank metadata lookups.

## Understanding Standby-Replay

With `activeStandby: true` in Rook (or `allow_standby_replay true` in Ceph), each active rank has a dedicated warm standby that continuously replays the journal. This reduces failover time from tens of seconds to under a second:

```bash
ceph fs set myfs allow_standby_replay true
```

Check standby-replay MDS:

```bash
ceph mds stat
```

Output showing standby-replay:

```text
myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
```

## Summary

MDS ranks are active slots that serve CephFS namespace subtrees. Use `ceph fs set myfs max_mds <n>` to scale active ranks, or configure `activeCount` in Rook's `CephFilesystem` CRD. Pin directories to specific ranks for performance locality. Enable `activeStandby: true` in Rook for fast failover via standby-replay daemons.
