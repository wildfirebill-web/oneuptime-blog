# How to Configure Device Health Monitoring in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Device Health, Storage, Kubernetes, Monitoring

Description: Learn how to enable and configure Ceph device health monitoring to predict disk failures and automate preventive OSD replacement.

---

## What Is Device Health Monitoring in Ceph

Ceph includes a built-in device health monitoring subsystem that collects S.M.A.R.T. data from underlying storage devices. By analyzing this data, Ceph can predict when a drive is likely to fail and mark it as being replaced before data loss occurs. In Rook-managed clusters, this feature integrates with Kubernetes to automate OSD lifecycle management.

## Enabling the Device Health Module

The device health module is managed through the Ceph manager. Enable it with:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mgr module enable devicehealth
```

Verify it is running:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mgr module ls | grep devicehealth
```

## Configuring Health Check Intervals

Once enabled, configure how often Ceph polls S.M.A.R.T. data from devices:

```bash
# Check devices every 24 hours (86400 seconds)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set global device_health_scrape_frequency 86400

# Retain health metrics history for 86400 seconds per sample
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set global device_health_metrics_max_age 8640000
```

## Setting the Failure Prediction Mode

Ceph can use a local prediction model or an external one. Set the prediction mode:

```bash
# Use the built-in linear regression model
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set global device_health_prediction_mode local

# Available values: none, local
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config get global device_health_prediction_mode
```

## Configuring Automatic OSD Marking

When a device is predicted to fail within a certain number of days, Ceph can automatically mark the corresponding OSD out. Configure this threshold:

```bash
# Mark OSD out if failure predicted within 14 days
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set global device_health_target_daemon osd

# Enable automatic marking of predicted-failure OSDs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set global device_health_enable_monitoring true

# Days before predicted failure to mark OSD out
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set global device_health_mark_out_threshold 14
```

## Manually Collecting Device Health Data

You can trigger an immediate collection of S.M.A.R.T. data without waiting for the scheduled interval:

```bash
# Collect health data for all devices
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph device scrape-health-metrics

# Collect for a specific device
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph device scrape-health-metrics <device-id>
```

## Viewing Device Health Data

List all tracked devices and their health status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph device ls
```

View detailed health history for a specific device:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph device get-health-metrics <device-id>
```

Check the current prediction for a device:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph device predict-life-expectancy <device-id>
```

## Rook CephCluster Configuration

You can also configure device health monitoring declaratively through the Rook CephCluster resource:

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
      osd:
        disabled: false
        interval: 60s
      status:
        disabled: false
        interval: 60s
```

## Alerts for Predicted Failures

Ceph raises health warnings when a device is predicted to fail. Check for these alerts:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail | grep -i device
```

You will see messages like `DEVICE_HEALTH: 1 device(s) expected to fail soon` if predictions indicate imminent failure.

## Summary

Ceph device health monitoring uses S.M.A.R.T. data to predict disk failures before they cause data loss. By enabling the `devicehealth` manager module, configuring scrape intervals, and setting automatic OSD marking thresholds, you can build a proactive storage management workflow that replaces failing drives with minimal disruption to the cluster.
