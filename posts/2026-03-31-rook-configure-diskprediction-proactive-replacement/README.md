# How to Configure DiskPrediction for Proactive Disk Replacement

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, DiskPrediction, Device Health, Storage, Predictive Maintenance

Description: Learn how to configure Ceph DiskPrediction and device health module settings to enable automatic disk failure prediction and proactive OSD replacement workflows.

---

## What Is DiskPrediction in Ceph?

DiskPrediction is the capability within the Ceph `devicehealth` manager module that uses SMART disk attributes and machine learning models to predict drive failures before they occur. It integrates with either Ceph's local prediction models or an external DiskPrediction cloud service.

## Prerequisites

Ensure the device health module is enabled and your OSDs have access to SMART data:

```bash
ceph mgr module enable devicehealth
ceph device ls
```

## Configuring Local Health Prediction

For most environments, local health prediction uses SMART data collected directly from drives:

```bash
# Set the SMART data collection frequency (seconds)
ceph config set mgr mgr/devicehealth/scrape_frequency 86400

# Set prediction horizon (days to look ahead)
ceph config set mgr mgr/devicehealth/prediction_mode local
```

Trigger an immediate health check:

```bash
ceph device check-health
```

## Configuring DiskPrediction Cloud (Optional)

Ceph supports integration with a DiskPrediction cloud service for enhanced ML-based predictions:

```bash
ceph config set mgr mgr/diskprediction_cloud/diskprediction_server "service.endpoint.com"
ceph config set mgr mgr/diskprediction_cloud/diskprediction_port 8086
ceph config set mgr mgr/diskprediction_cloud/diskprediction_user "your-user"
ceph config set mgr mgr/diskprediction_cloud/diskprediction_password "your-password"
ceph mgr module enable diskprediction_cloud
```

## Setting Failure Thresholds

Configure when Ceph should warn and act on predicted failures:

```bash
# Warn about drives predicted to fail within 14 days
ceph config set mgr mgr/devicehealth/warn_threshold 14

# Automatically mark OSDs out when predicted to fail within 7 days
ceph config set mgr mgr/devicehealth/mark_out_threshold 7
```

Verify current thresholds:

```bash
ceph config get mgr mgr/devicehealth/warn_threshold
ceph config get mgr mgr/devicehealth/mark_out_threshold
```

## Enabling Self-Healing

With `self_heal` enabled, Ceph automatically marks predicted-failure OSDs as `out` to trigger data migration:

```bash
ceph config set mgr mgr/devicehealth/self_heal true
```

Check what the device health module will do with a specific device:

```bash
ceph device info <device-id>
ceph health detail | grep DEVICE_HEALTH
```

## Monitoring Prediction Status

View prediction results for all devices:

```bash
ceph device ls --format json | python3 -c "
import sys, json
devices = json.load(sys.stdin)
for d in devices:
    dev_id = d.get('devid', 'N/A')
    life_min = d.get('life_expectancy_min', 'N/A')
    life_max = d.get('life_expectancy_max', 'N/A')
    print(f'{dev_id}: {life_min} to {life_max}')
"
```

Prometheus metrics for device health:

```
ceph_device_health_forecast_score
```

## Proactive Replacement Workflow in Rook

When a device is flagged for replacement:

1. Verify the flagged device:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph device info <device-id>
```

2. Confirm the OSD is marked out and data is migrated:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd tree | grep out
```

3. Replace the physical disk and let Rook reprovision the OSD automatically.

## Summary

DiskPrediction in Ceph combines SMART data collection with predictive models to identify at-risk drives before they fail. Configuring warn and mark-out thresholds automates the protection response. With `self_heal` enabled, Ceph proactively moves data off predicted-failure devices, giving operators a window to replace hardware before any data loss occurs.
