# How to Verify Health Before and After Rook Upgrades

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Health, Upgrade, Verification

Description: Learn the essential health checks to run before and after Rook-Ceph upgrades to ensure cluster stability and catch issues early.

---

Health verification before and after Rook upgrades is essential for safe upgrades. Pre-upgrade checks confirm the cluster can tolerate the disruption of a rolling upgrade. Post-upgrade checks confirm everything is functioning correctly with the new version. Skipping these checks risks missing pre-existing issues that the upgrade exacerbates.

## Pre-Upgrade Health Checks

Run all checks and document the baseline state before beginning any upgrade.

### Cluster-Level Health

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

The cluster must show `HEALTH_OK` before upgrading. If it shows `HEALTH_WARN` or `HEALTH_ERR`, investigate and resolve all issues first.

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

This shows the specific reason for any health warnings. Address all `HEALTH_ERR` items. Some `HEALTH_WARN` items (like `application not enabled on pool`) may be acceptable, but document any warnings you choose to proceed with.

### OSD Status

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd stat
```

```text
6 osds: 6 up (since 5d), 6 in (since 5d)
```

All OSDs must be `up` and `in`. If any OSD is `down` or `out`, resolve it before upgrading.

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd df
```

Check OSD usage. No OSD should be near-full (above 85% usage) before upgrading.

### Placement Group Status

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg stat
```

```text
192 pgs: 192 active+clean; 5.4 GiB data, 16 GiB used, 94 GiB / 110 GiB avail
```

All PGs must be `active+clean`. PGs in `degraded`, `undersized`, `peering`, or `incomplete` states indicate the cluster is not ready for an upgrade.

### Monitor Quorum

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph quorum_status --format json-pretty | python3 -m json.tool
```

Verify all monitors are in the quorum list.

### Kubernetes Pod Status

```bash
kubectl -n rook-ceph get pods | grep -v Running | grep -v Completed
```

All Rook pods should be Running. Any CrashLoopBackOff or Error pods need investigation.

## Recording Baseline State

Save the pre-upgrade state for comparison:

```bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BASELINE_DIR="rook-upgrade-baseline-${TIMESTAMP}"
mkdir -p "$BASELINE_DIR"

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status > "${BASELINE_DIR}/ceph-status.txt"
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd df > "${BASELINE_DIR}/osd-df.txt"
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph pg stat > "${BASELINE_DIR}/pg-stat.txt"
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph versions > "${BASELINE_DIR}/versions.txt"
kubectl -n rook-ceph get pods -o wide > "${BASELINE_DIR}/pods.txt"
kubectl -n rook-ceph get deployment -o wide > "${BASELINE_DIR}/deployments.txt"

echo "Baseline saved to ${BASELINE_DIR}"
```

## During Upgrade Monitoring

While the upgrade is in progress, monitor the cluster continuously:

```bash
watch -n 15 "kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status 2>/dev/null | head -20"
```

Also watch pod restarts:

```bash
kubectl -n rook-ceph get pods -w
```

It is normal to see brief `HEALTH_WARN` states during rolling restarts of daemons. This should resolve within minutes of each daemon restarting.

## Post-Upgrade Health Checks

Run the same checks as pre-upgrade and compare against the baseline.

### Verify Cluster Health

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

### Verify All Daemons on New Version

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph versions
```

All mon, mgr, osd, and mds daemons should show the new Ceph version.

### Verify Pod Images

```bash
kubectl -n rook-ceph get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}' | sort
```

### Storage Functionality Test

After upgrading, test that storage still works by writing and reading data:

```bash
# Test block storage
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd -p replicapool create test-image --size 10M
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd -p replicapool info test-image
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd -p replicapool rm test-image

# Test object storage
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p replicapool put upgrade-test /etc/hostname
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p replicapool rm upgrade-test
```

## Summary

Verify Rook-Ceph upgrade success by running a complete set of health checks before and after the upgrade, saving a baseline snapshot for comparison, monitoring cluster health continuously during the upgrade, and performing functional storage tests after completion. The cluster must show `HEALTH_OK` with all OSDs up and all PGs `active+clean` before starting any upgrade. Post-upgrade, confirm all daemons report the new version using `ceph versions`.
