# How to Upgrade Ceph Clusters in Stretch Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Stretch Mode, Upgrade, Operation

Description: Step-by-step guide to safely upgrading Ceph clusters running in stretch mode with minimal downtime and zero data loss.

---

## Upgrade Considerations for Stretch Mode

Upgrading a Ceph cluster in stretch mode requires extra care because you must maintain quorum throughout the upgrade. Upgrading monitors or OSDs on both sites simultaneously can break quorum and cause a cluster-wide outage.

## Pre-Upgrade Checklist

Before starting the upgrade, verify the cluster is healthy:

```bash
ceph status
ceph health detail
ceph osd stat
ceph mon stat
```

Ensure all PGs are active+clean:

```bash
ceph pg stat | grep "active+clean"
```

Set conservative flags to prevent rebalancing during the upgrade:

```bash
ceph osd set noout
ceph osd set noscrub
ceph osd set nodeep-scrub
```

## Step 1 - Upgrade the Arbiter Monitor

Start with the arbiter since it has no OSDs and minimal impact on cluster operations:

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 --daemon-types mon --hosts mon-arbiter
```

Monitor the upgrade:

```bash
ceph orch upgrade status
```

Wait for the arbiter to rejoin quorum:

```bash
ceph mon stat
```

## Step 2 - Upgrade Site A Monitors

Upgrade the two monitors on site A one at a time:

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 --daemon-types mon --hosts mon-dc1a
```

Wait for quorum:

```bash
ceph quorum_status
```

Then upgrade the second monitor on site A:

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 --daemon-types mon --hosts mon-dc1b
```

## Step 3 - Upgrade Site B Monitors

Repeat for site B monitors:

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 --daemon-types mon --hosts mon-dc2a
ceph quorum_status
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 --daemon-types mon --hosts mon-dc2b
```

## Step 4 - Upgrade Managers

Upgrade MGR daemons:

```bash
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 --daemon-types mgr
```

## Step 5 - Upgrade OSDs by Site

Upgrade OSDs on site A first, then site B. This maintains data accessibility from at least one site:

```bash
# Upgrade site A OSDs
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 \
  --daemon-types osd --hosts host-dc1a,host-dc1b

# Wait for PGs to recover
watch ceph pg stat

# Upgrade site B OSDs
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0 \
  --daemon-types osd --hosts host-dc2a,host-dc2b
```

## Step 6 - Post-Upgrade Verification

After all daemons are upgraded, clear the maintenance flags:

```bash
ceph osd unset noout
ceph osd unset noscrub
ceph osd unset nodeep-scrub
```

Verify the cluster version and health:

```bash
ceph version
ceph status
ceph orch ps | grep -v "running"
```

## Summary

Upgrading Ceph clusters in stretch mode requires a sequential per-site approach starting with the arbiter monitor. By upgrading monitors and OSDs site by site and waiting for quorum and PG health at each step, you maintain continuous availability throughout the upgrade process. Setting maintenance flags before starting prevents unnecessary rebalancing during the rolling upgrade.
