# How to Configure Scrub Priority and Throttling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Scrubbing, Performance, Throttling, Kubernetes, OSD

Description: Learn how to configure Ceph scrub priority and I/O throttling to control the performance impact of scrubbing operations on production workloads.

---

## Why Scrub Throttling Matters

Ceph scrubbing reads data from OSDs to verify integrity. Without throttling, a deep scrub can consume the majority of an OSD's I/O bandwidth, causing latency spikes for production workloads. Proper throttling ensures scrubs run in the background without impacting client performance.

## Scrub Load Control Parameters

Key parameters that control scrub I/O impact:

| Parameter | Default | Description |
|---|---|---|
| `osd_scrub_sleep` | 0 | Sleep between object scrubs (seconds) |
| `osd_max_scrubs` | 1 | Maximum simultaneous scrubs per OSD |
| `osd_scrub_chunk_min` | 5 | Minimum objects per scrub iteration |
| `osd_scrub_chunk_max` | 25 | Maximum objects per scrub iteration |
| `osd_deep_scrub_sleep` | 0 | Sleep between deep scrub object reads |

## Setting Scrub Sleep to Throttle I/O

Add sleep intervals between scrub object reads to reduce I/O pressure:

```bash
# Add 0.1 second pause between each object during shallow scrubs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_sleep 0.1

# Add 0.5 second pause for deep scrubs (more I/O intensive)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_deep_scrub_sleep 0.5

# Verify current settings
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config get osd osd_scrub_sleep
```

## Limiting Concurrent Scrubs

Reduce the number of simultaneous scrubs per OSD:

```bash
# Only 1 scrub at a time per OSD (default and recommended)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_max_scrubs 1

# Verify
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config get osd osd_max_scrubs
```

## Adjusting Scrub Chunk Size

Smaller chunks reduce burst I/O but extend total scrub duration:

```bash
# Smaller chunks for lighter impact
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_chunk_min 1

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_chunk_max 5
```

## Setting Scrub Priority

When both client I/O and scrubbing compete for OSD resources, control the priority:

```bash
# Set scrub I/O priority (class_id 0=idle, 2=best-effort, 6=high)
# This uses the Linux I/O scheduler priority when available
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_disk_thread_ioprio_class idle

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_disk_thread_ioprio_priority 7
```

## Recommended Throttling Profiles

For production clusters with sensitive workloads:

```bash
# Conservative: minimal scrub impact
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_sleep 0.2
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_deep_scrub_sleep 1.0
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_chunk_max 5
```

For dev/test clusters where speed is preferred:

```bash
# Aggressive: scrub as fast as possible
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_sleep 0
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_deep_scrub_sleep 0
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_scrub_chunk_max 25
```

## Monitoring Scrub Impact

```bash
# Watch OSD performance during scrubbing
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch -n 2 "ceph osd perf | sort -k4 -rn | head -10"
```

## Summary

Ceph scrub throttling controls the balance between data integrity verification speed and production workload impact. The most effective throttle is `osd_scrub_sleep` and `osd_deep_scrub_sleep`, which add pauses between object reads. Combine sleep settings with smaller chunk sizes for the least bursty scrub behavior. For production clusters, use conservative throttling during business hours and remove throttling during maintenance windows when scrubs can run at full speed.
