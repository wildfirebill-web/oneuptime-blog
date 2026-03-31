# How to Remove Hosts with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Host, Decommission, Cluster

Description: Safely remove a host from a cephadm-managed Ceph cluster by draining services, removing OSDs, and deregistering the node.

---

## When to Remove a Host

You may need to remove a Ceph host for:

- Hardware decommission or replacement
- Cluster downsizing
- Moving to a different failure domain
- Host OS reinstall or repurposing

The key principle is: never remove a host abruptly. Always drain services and allow data recovery first.

## Step 1: Check What Is Running on the Host

```bash
ceph orch ps --hostname node4
```

Note all running daemons: OSDs, monitors, RGW, MDS, etc.

## Step 2: Remove OSDs First

For each OSD on the host, drain and remove it:

```bash
# Remove all OSDs on a specific host
ceph orch osd rm --host node4 --zap

# Or remove specific OSD IDs
ceph osd out 12 13 14
ceph orch osd rm 12 13 14 --zap
```

Monitor the OSD removal and data migration:

```bash
ceph osd rm status
ceph status
```

Wait until the cluster returns to `HEALTH_OK` before continuing.

## Step 3: Drain All Other Services

Remove all non-OSD daemons from the host:

```bash
ceph orch host drain node4
```

This removes monitors, RGW instances, MDS daemons, and other services. Check progress:

```bash
ceph orch ps --hostname node4
```

Wait until no daemons remain.

## Step 4: Remove the Host

Once all services are drained:

```bash
ceph orch host rm node4
```

If the host is offline and cannot be reached:

```bash
ceph orch host rm node4 --force --offline
```

## Step 5: Verify Removal

```bash
ceph orch host ls
ceph status
```

Confirm the host no longer appears and the cluster is healthy.

## Step 6: Clean Up the Node (Optional)

If you plan to reuse the host, clean the Ceph installation:

```bash
# Run on the decommissioned node
sudo cephadm rm-cluster --fsid <cluster-fsid> --force
```

Find the FSID:

```bash
ceph fsid
```

## Handling a Failed/Unreachable Host

If the host is completely dead:

```bash
# Mark all OSDs on the host as out
ceph osd out osd.12 osd.13

# After data migration completes, remove the OSDs
ceph osd purge 12 --yes-i-really-mean-it
ceph osd purge 13 --yes-i-really-mean-it

# Force-remove the host record
ceph orch host rm node4 --force --offline
```

## Summary

To safely remove a cephadm-managed host, first drain OSDs with `ceph orch osd rm --zap`, wait for data migration to complete, then use `ceph orch host drain` to remove remaining services, and finally `ceph orch host rm` to deregister the node. Always verify cluster health at each step before proceeding.
