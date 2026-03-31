# How to Configure Health Check Settings for OSDs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, OSD, HealthCheck, Reliability

Description: Tune OSD health check intervals and timeouts in Rook so the operator detects disk failures quickly and initiates recovery without triggering false-positive OSD removals.

---

## Why OSD Health Checks Need Tuning

Ceph marks an OSD as down and starts recovery when the OSD fails to respond for a set period. Rook adds a second layer: the operator watches OSD pod health and can evict or replace OSDs that remain unhealthy.

Too-aggressive health check timeouts cause spurious OSD removal on slow disks or network blips, leading to unnecessary data rebalancing. Too-long timeouts delay recovery when a disk genuinely fails. Finding the right balance is critical for maintaining cluster performance.

## Configuring OSD Health Checks

Set the `osd` block under `healthCheck.daemonHealth` in your `CephCluster` CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  healthCheck:
    daemonHealth:
      osd:
        disabled: false
        interval: 60s
        timeout: 600s
```

`interval` is how often the operator polls OSD health. `timeout` is how long an OSD can remain unhealthy before operator-level remediation begins.

## OSD Removal vs Ceph's Own osd.down Mechanism

Ceph internally marks an OSD `down` after `mon_osd_report_timeout` seconds (default 900s). Rook's `timeout` triggers different behavior: it signals that the OSD pod itself is unhealthy and may need to be rescheduled. These two timeouts operate independently.

For most production clusters, align the Rook timeout with or slightly below Ceph's internal timeout:

```yaml
healthCheck:
  daemonHealth:
    osd:
      interval: 60s
      timeout: 600s   # slightly below Ceph's default 900s
```

## High-Latency Storage Environments

On clusters with slow HDDs or heavily loaded nodes, OSD responses can be delayed. Increase the timeout to avoid false positives:

```yaml
healthCheck:
  daemonHealth:
    osd:
      interval: 120s
      timeout: 900s
```

## NVMe Clusters Requiring Fast Detection

For all-NVMe clusters where disk failures are clean and abrupt, reduce the timeout for faster automated recovery:

```yaml
healthCheck:
  daemonHealth:
    osd:
      interval: 30s
      timeout: 180s
```

## Monitoring OSD Health Manually

Check OSD status through the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd stat
```

List OSDs that are currently down:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree | grep -E "down|out"
```

Watch OSD pod restarts as an indicator of recurring issues:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-osd \
  -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,RESTARTS:.status.containerStatuses[0].restartCount'
```

## Disabling OSD Health Checks During Maintenance

Before draining a node or replacing disks, disable OSD health checks to prevent the operator from reacting to expected OSD absences:

```yaml
healthCheck:
  daemonHealth:
    osd:
      disabled: true
```

Restore after the maintenance window closes.

## Summary

OSD health check configuration in Rook requires balancing the speed of failure detection against the risk of spurious remediation. Match your timeout values to your disk technology (fast NVMe vs slow HDD) and network reliability. Always disable health checks during planned maintenance to avoid triggering recovery operations that compete with your maintenance work.
