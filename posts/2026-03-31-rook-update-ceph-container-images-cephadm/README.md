# How to Update Ceph Container Images with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Upgrade, Container, Image, Maintenance

Description: Upgrade Ceph daemons to a new version by updating container images with cephadm, including pre-upgrade health checks and rollback procedures.

---

## How cephadm Upgrades Work

cephadm upgrades Ceph by replacing the container image used for each daemon. It upgrades in a controlled order: monitors first, then managers, then OSDs, then other services. Each daemon is restarted one at a time, allowing the cluster to remain operational throughout the process.

## Step 1: Pre-Upgrade Health Check

Before upgrading, ensure the cluster is healthy:

```bash
ceph status
ceph health detail
```

All PGs should be `active+clean` and no OSDs should be `down`. Resolve any issues before proceeding.

## Step 2: Check Current Versions

```bash
ceph versions
```

This shows daemon counts per Ceph version, useful for verifying upgrade progress.

## Step 3: Check Upgrade Path

Ceph supports upgrades one major version at a time:

- Octopus (15) -> Pacific (16) -> Quincy (17) -> Reef (18)

Do not skip major versions.

## Step 4: Set the New Container Image

```bash
ceph orch upgrade start \
  --image quay.io/ceph/ceph:v18.2.0
```

Or for a specific major version target:

```bash
ceph orch upgrade start --ceph-version 18.2.0
```

## Step 5: Monitor Upgrade Progress

```bash
ceph orch upgrade status
```

Sample output:

```
target_image: quay.io/ceph/ceph:v18.2.0
upgrading: true
progress: 8/24 daemons upgraded
```

Watch in real time:

```bash
ceph -w | grep upgrade
```

## Step 6: Verify After Upgrade

```bash
ceph versions
ceph status
ceph health detail
```

All daemons should report the new version.

## Pausing and Resuming an Upgrade

```bash
# Pause
ceph orch upgrade pause

# Resume
ceph orch upgrade resume
```

## Rolling Back (If Issues Arise)

If the upgrade causes problems, stop and revert to the old image:

```bash
ceph orch upgrade stop

# Set the old image
ceph config set global container_image quay.io/ceph/ceph:v17.2.7

# Restart affected services
ceph orch restart mon
ceph orch restart mgr
```

## Upgrading Monitoring Stack

The Prometheus and Grafana images are upgraded separately:

```bash
ceph orch apply prometheus --image quay.io/prometheus/prometheus:v2.47.0
ceph orch apply grafana --image quay.io/grafana/grafana:10.1.0
```

## Summary

cephadm upgrades proceed by running `ceph orch upgrade start --image` with the new image tag, upgrading daemons in safe order (mon, mgr, OSD, others). Monitor progress with `ceph orch upgrade status` and use `ceph versions` to confirm completion. Always run a full health check before starting and have the old image tag ready for a rollback if needed.
