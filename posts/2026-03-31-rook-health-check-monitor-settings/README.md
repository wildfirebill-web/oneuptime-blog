# How to Configure Health Check Settings for Monitors in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Monitor, HealthCheck, Reliability

Description: Configure health check intervals and timeouts for Ceph monitor daemons in Rook to control how quickly the operator detects and responds to monitor failures.

---

## How Rook Monitors Health

The Rook operator continuously watches the health of all Ceph components. For monitors specifically, it queries the Ceph cluster status and checks whether each monitor is in quorum. The `healthCheck` section of the `CephCluster` CRD lets you tune how frequently these checks run and how long a failure must persist before the operator takes remediation action.

## Configuring Monitor Health Checks

Add the `healthCheck` block under `spec` in your `CephCluster` resource:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  healthCheck:
    daemonHealth:
      mon:
        disabled: false
        interval: 45s
        timeout: 600s
```

`interval` controls how often Rook probes the monitor's health. `timeout` defines how long a monitor can remain unhealthy before the operator attempts to failover or restart it.

## Understanding the Timeout

The `timeout` for monitors is the total time the operator waits before declaring a monitor failed and triggering remediation. The default is 600 seconds (10 minutes). This conservative default prevents flapping during transient network issues.

For clusters with reliable network infrastructure, reducing this value allows faster failover:

```yaml
healthCheck:
  daemonHealth:
    mon:
      interval: 30s
      timeout: 300s
```

For clusters with occasional network instability (cross-datacenter links, WiFi-connected nodes in a lab), increase the timeout to avoid false-positive failovers:

```yaml
healthCheck:
  daemonHealth:
    mon:
      interval: 60s
      timeout: 900s
```

## Disabling Health Checks Temporarily

During planned maintenance, disable monitor health checks to prevent the operator from reacting to temporary disruptions:

```yaml
healthCheck:
  daemonHealth:
    mon:
      disabled: true
```

Re-enable them after maintenance completes by setting `disabled: false`.

## Monitor Health in the Ceph Status

The operator also surfaces health via the `CephCluster` status condition. Watch cluster health in real time:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph -o jsonpath='{.status.ceph.health}'
```

Detailed monitor quorum status is available through the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
```

Expected output for a healthy three-monitor cluster:

```text
e3: 3 mons at {a=[v2:10.0.0.1:3300/0,v1:10.0.0.1:6789/0],
               b=[v2:10.0.0.2:3300/0,v1:10.0.0.2:6789/0],
               c=[v2:10.0.0.3:3300/0,v1:10.0.0.3:6789/0]},
election epoch 12, leader 0 a, quorum 0,1,2 a,b,c
```

## Correlating with Kubernetes Probes

Monitor health checks at the Rook operator level are separate from the Kubernetes liveness probes on the monitor pods themselves. Liveness probes restart individual containers; Rook health checks make decisions about quorum and trigger higher-level remediation like replacing a monitor on a failed node.

Both layers are needed for complete health management.

## Summary

Tuning monitor health check intervals and timeouts in Rook lets you balance responsiveness against stability. Shorter intervals and timeouts improve mean time to recovery for fast networks; longer values reduce spurious failovers on slower or less reliable links. Always re-enable health checks after maintenance to ensure the operator resumes protecting monitor quorum.
