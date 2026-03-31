# How to Fix OSD_NO_DOWN_OUT_INTERVAL Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, OSD

Description: Learn how to resolve the OSD_NO_DOWN_OUT_INTERVAL health warning in Ceph by configuring the mon_osd_down_out_interval setting correctly.

---

## What Is OSD_NO_DOWN_OUT_INTERVAL

The `OSD_NO_DOWN_OUT_INTERVAL` health warning appears when the `mon_osd_down_out_interval` configuration option is set to zero or a very low value. This setting controls how long Ceph waits before marking a down OSD as "out" and redistributing its data. When set to zero, Ceph immediately starts rebalancing data the moment an OSD goes down, which can cause massive unnecessary rebalancing during transient failures like network blips or brief node reboots.

Check cluster health:

```bash
ceph health detail
```

Sample output:

```text
HEALTH_WARN mon_osd_down_out_interval is 0
[WRN] OSD_NO_DOWN_OUT_INTERVAL: mon_osd_down_out_interval is 0
    Having a non-zero mon_osd_down_out_interval avoids short-lived OSD outages causing large amounts of unnecessary data migration
```

## Why This Is Dangerous

With `mon_osd_down_out_interval` set to zero, even a 30-second network timeout causes Ceph to begin moving all the data that was on that OSD to other OSDs. If the OSD comes back online moments later, Ceph must reverse all that migration. On large clusters, this can saturate disk I/O and cluster network bandwidth for hours.

## Checking Current Value

Inspect the current setting:

```bash
ceph config get mon mon_osd_down_out_interval
```

A value of `0` triggers the health warning.

## Fixing the Warning

Set a reasonable interval. The default and recommended value is 600 seconds (10 minutes):

```bash
ceph config set mon mon_osd_down_out_interval 600
```

For clusters where faster rebalancing is acceptable (e.g., test environments), a value of 300 seconds is common:

```bash
ceph config set mon mon_osd_down_out_interval 300
```

## Verifying the Fix

Confirm the value was applied:

```bash
ceph config get mon mon_osd_down_out_interval
```

Expected output:

```text
600
```

Check cluster health:

```bash
ceph health
```

Expected:

```text
HEALTH_OK
```

## Rook-Specific Configuration

In a Rook-managed cluster, you can set this via the Rook `CephCluster` CR using the `cephConfig` section:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephConfig:
    mon:
      mon_osd_down_out_interval: "600"
```

Apply the change:

```bash
kubectl apply -f cephcluster.yaml
```

Rook will propagate the configuration to all monitor daemons automatically.

## Related Settings

While fixing this, also verify the companion setting `mon_osd_down_out_subtree_limit`, which prevents Ceph from marking too many OSDs out at once:

```bash
ceph config get mon mon_osd_down_out_subtree_limit
```

The default `host` value means Ceph will not mark an entire host's OSDs as out, preventing catastrophic data loss during host failures.

## Summary

The `OSD_NO_DOWN_OUT_INTERVAL` warning indicates `mon_osd_down_out_interval` is zero, which causes excessive data rebalancing during transient OSD failures. Fix it by setting the interval to at least 300-600 seconds using `ceph config set` or via the Rook `CephCluster` CRD's `cephConfig` section.
