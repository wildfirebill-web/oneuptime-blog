# How to Use the DiskPrediction Module in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, DiskPrediction, Predictive, Storage

Description: Enable and configure the Ceph DiskPrediction module to detect failing drives early using ML-based health scoring and SMART data analysis.

---

The Ceph DiskPrediction module uses machine learning models to analyze OSD health data and SMART disk metrics, predicting drive failures before they cause data loss.

## DiskPrediction Module Variants

Ceph offers two DiskPrediction modules:

- **diskprediction_local** - Uses a local ML model (no external dependencies)
- **diskprediction_cloud** - Sends data to Prophetstor cloud service for predictions

## Enabling the Local Module

```bash
# Enable local disk prediction
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph mgr module enable diskprediction_local

# Verify it is active
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph mgr module ls | grep diskprediction
```

## Configuring Prediction Parameters

```bash
# Set data collection interval (seconds, default: 600)
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph config set mgr mgr/diskprediction_local/predict_interval 600

# Set prediction mode
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph config set mgr mgr/diskprediction_local/predict_base_dir /var/lib/ceph/disk_predictions
```

## Viewing Disk Health Predictions

```bash
# Get prediction for all OSDs
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph device ls

# Get health prediction for a specific device
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph device get-predicted-life-expectancy DEVICE_ID
```

Sample output:

```bash
{
    "device_id": "SAMSUNG_MZ7LH960HAJR_S3EWNX0M123456",
    "near_death": false,
    "life_expectancy_min": "2026-12-01T00:00:00.000000",
    "life_expectancy_max": "2027-06-01T00:00:00.000000"
}
```

## Checking SMART Data

```bash
# List devices with SMART info
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph device ls --format json | python3 -m json.tool

# Get SMART metrics for a device
kubectl -n rook-ceph exec -it deploy/rook-ceph-mgr-a -- \
  ceph device info DEVICE_ID
```

## Setting Up Proactive OSD Replacement

Combine DiskPrediction with Prometheus alerting:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ceph-disk-health
  namespace: rook-ceph
spec:
  groups:
  - name: disk-prediction
    rules:
    - alert: CephDeviceNearDeath
      expr: ceph_device_health_score < 0.2
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Ceph OSD device at risk of failure"
        description: "Device {{ $labels.devid }} has health score {{ $value }}"
```

## Summary

The Ceph DiskPrediction local module uses ML models to score OSD disk health and predict failure timelines. Enable it with `ceph mgr module enable diskprediction_local`, then query predictions with `ceph device get-predicted-life-expectancy`. Combine with Prometheus alerting to trigger proactive OSD replacement before actual failures occur.
