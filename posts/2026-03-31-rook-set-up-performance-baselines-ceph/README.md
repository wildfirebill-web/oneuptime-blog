# How to Set Up Performance Baselines for Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Performance, Benchmarking, Monitoring

Description: Learn how to establish performance baselines for your Ceph cluster by running standardized benchmarks and recording key metrics to detect future regressions and verify changes.

---

## Why Baselines Matter

A performance baseline is a recorded snapshot of your cluster's performance under known conditions. Without a baseline, it is impossible to answer questions like "is this cluster slower than before?" or "did this upgrade improve or degrade performance?" Baselines enable objective performance regression detection.

## When to Create Baselines

Capture baselines at these key moments:

- After initial cluster deployment and validation
- After adding new nodes or OSDs
- Before and after Ceph or Rook version upgrades
- After changing pool settings (replication factor, erasure coding, compression)
- After hardware changes (new disk type, NIC upgrade)

## Run rados bench for Object Storage Baseline

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Create a baseline pool
  ceph osd pool create baseline-test 32

  # Sequential write benchmark (60 seconds)
  rados bench -p baseline-test 60 write --no-cleanup 2>&1 | tee /tmp/rados-write-baseline.txt

  # Sequential read benchmark
  rados bench -p baseline-test 60 seq 2>&1 | tee /tmp/rados-read-seq-baseline.txt

  # Random read benchmark
  rados bench -p baseline-test 60 rand 2>&1 | tee /tmp/rados-read-rand-baseline.txt

  # Cleanup
  rados -p baseline-test cleanup
  ceph osd pool delete baseline-test baseline-test --yes-i-really-really-mean-it
"
```

## Run rbd bench for Block Storage Baseline

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Create a test image
  rbd create --pool rbd --image baseline-test --size 10G

  # Sequential write
  rbd bench --io-type write --io-size 4096 --io-threads 16 \
    --io-total 1073741824 rbd/baseline-test 2>&1 | tee /tmp/rbd-write-baseline.txt

  # Random read
  rbd bench --io-type read --io-size 4096 --io-threads 16 \
    --io-total 1073741824 rbd/baseline-test 2>&1 | tee /tmp/rbd-read-baseline.txt

  # Cleanup
  rbd rm rbd/baseline-test
"
```

## Record Key Prometheus Metrics

Capture baseline values for key Prometheus metrics during normal operation:

```bash
#!/bin/bash
# capture-prometheus-baseline.sh
PROM_URL="http://prometheus:9090"

echo "=== Ceph Baseline Metrics $(date) ===" > baseline-prometheus.txt

for metric in \
  "avg(ceph_osd_apply_latency_ms)" \
  "avg(ceph_osd_commit_latency_ms)" \
  "sum(rate(ceph_osd_op_w[5m]))" \
  "sum(rate(ceph_osd_op_r[5m]))" \
  "ceph_cluster_total_used_bytes / ceph_cluster_total_bytes"; do

  VALUE=$(curl -s "$PROM_URL/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$metric'))")" | jq -r '.data.result[0].value[1]')
  echo "$metric = $VALUE" >> baseline-prometheus.txt
done
```

## Store Baselines in Git

Commit all baseline files to a Git repository:

```bash
mkdir -p baselines/$(date +%Y-%m-%d)-rook-1.13.2
cp /tmp/rados-*.txt baselines/$(date +%Y-%m-%d)-rook-1.13.2/
cp baseline-prometheus.txt baselines/$(date +%Y-%m-%d)-rook-1.13.2/
git add baselines/
git commit -m "Add performance baselines for Rook 1.13.2 on $(date +%Y-%m-%d)"
```

## Compare Against Baseline

After an upgrade or change, run the same benchmarks and compare:

```bash
diff baselines/2026-03-01-rook-1.13.1/rados-write-baseline.txt \
     baselines/2026-03-31-rook-1.13.2/rados-write-baseline.txt
```

Focus on the `Bandwidth (MB/s)`, `Average IOPS`, and `Average Latency` lines in the rados bench output.

## Summary

Setting up Ceph performance baselines involves running standardized `rados bench` and `rbd bench` tests, capturing Prometheus metric values, and storing results in version-controlled files. By running the same benchmarks before and after every significant change, you can objectively measure performance regressions or improvements and make data-driven decisions about configuration tuning and hardware upgrades.
