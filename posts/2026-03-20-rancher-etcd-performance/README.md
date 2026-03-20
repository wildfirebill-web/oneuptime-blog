# How to Optimize etcd Performance for Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, etcd, Performance, Kubernetes, Database Tuning

Description: Optimize etcd performance for Rancher clusters by tuning heartbeat intervals, compaction, storage, and monitoring key metrics to prevent etcd bottlenecks.

## Introduction

etcd is the backbone of every Kubernetes cluster, storing all cluster state. Poor etcd performance directly impacts API server responsiveness and cluster stability. This guide covers diagnosing and optimizing etcd performance for Rancher-managed clusters, including both the Rancher local cluster and downstream clusters.

## Prerequisites

- Access to etcd nodes (SSH or privileged pod access)
- kubectl with cluster-admin permissions
- etcdctl installed or accessible via pod

## Step 1: Check etcd Health

```bash
# Check etcd cluster health via kubectl
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key

# Check etcd cluster members
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key \
  -w table
```

## Step 2: Check etcd Database Size

```bash
# Check current DB size
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key \
  -w table

# Large database causes slow queries
# Compact and defragment if DB is large
# Compact to current revision
REVISION=$(kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key \
  -w json | jq '.[0].Status.header.revision')

kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl compact $REVISION \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key
```

## Step 3: Defragment etcd

```bash
# Defragment each member (do one at a time in production)
# Defragment the leader last
for endpoint in \
  https://etcd-0:2379 \
  https://etcd-1:2379 \
  https://etcd-2:2379; do

  echo "Defragmenting $endpoint..."
  etcdctl defrag \
    --endpoints=$endpoint \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/peer.crt \
    --key=/etc/kubernetes/pki/etcd/peer.key

  echo "Done. Waiting 30 seconds..."
  sleep 30
done
```

## Step 4: Tune etcd Parameters

```yaml
# RKE2 config for optimized etcd (on control plane nodes)
# /etc/rancher/rke2/config.yaml
etcd-arg:
  # Heartbeat interval (default: 100ms)
  # Increase for high-latency networks
  - "heartbeat-interval=300"
  # Election timeout (10x heartbeat)
  - "election-timeout=3000"
  # Auto compaction
  - "auto-compaction-mode=periodic"
  - "auto-compaction-retention=1h"
  # Quota (8GB to prevent space exceeded errors)
  - "quota-backend-bytes=8589934592"
  # Max request size (10MB)
  - "max-request-bytes=10485760"
  # Snapshot count before compaction trigger
  - "snapshot-count=10000"
```

## Step 5: Use Fast SSD Storage for etcd

```bash
# etcd is extremely I/O sensitive
# Benchmark your storage latency
# etcd needs <10ms write latency

# Test disk latency with fio
fio --rw=write --ioengine=sync \
    --fdatasync=1 \
    --directory=/var/lib/etcd \
    --size=22m \
    --bs=2300 \
    --name=etcd-test

# Check disk latency with dd
dd if=/dev/zero of=/var/lib/etcd/test bs=512 count=1000 oflag=dsync

# For RKE2, move etcd data directory to SSD
# Configure in /etc/rancher/rke2/config.yaml
# data-dir: /mnt/ssd/rancher/rke2
```

## Step 6: Monitor etcd Metrics

```yaml
# etcd-prometheus-alerts.yaml - Alert on etcd degradation
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: etcd
      rules:
        - alert: EtcdHighFsyncDurations
          expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "etcd WAL fsync p99 > 500ms"

        - alert: EtcdHighCommitDurations
          expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.25
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "etcd backend commit p99 > 250ms"

        - alert: EtcdDatabaseSizeLimitApproaching
          expr: (etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes) > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "etcd database size >80% of quota"
```

## Step 7: Schedule Regular Defragmentation

```yaml
# etcd-defrag-cronjob.yaml - Automated defragmentation
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-defrag
  namespace: kube-system
spec:
  schedule: "0 2 * * 0"  # Weekly at 2am Sunday
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
            - name: etcd-defrag
              image: rancher/hardened-etcd:v3.5.9
              command:
                - etcdctl
                - defrag
                - --endpoints=https://127.0.0.1:2379
              env:
                - name: ETCDCTL_CACERT
                  value: /etc/kubernetes/pki/etcd/ca.crt
                - name: ETCDCTL_CERT
                  value: /etc/kubernetes/pki/etcd/peer.crt
                - name: ETCDCTL_KEY
                  value: /etc/kubernetes/pki/etcd/peer.key
              volumeMounts:
                - name: etcd-certs
                  mountPath: /etc/kubernetes/pki/etcd
                  readOnly: true
          restartPolicy: OnFailure
          volumes:
            - name: etcd-certs
              hostPath:
                path: /etc/kubernetes/pki/etcd
```

## Conclusion

etcd performance is foundational to Kubernetes cluster health. Key optimizations include using NVMe/SSD storage with low fsync latency, tuning heartbeat intervals for your network conditions, enabling automatic compaction to prevent database growth, and scheduling regular defragmentation. Monitor etcd WAL fsync duration and backend commit latency as primary SLIs. For Rancher deployments, ensuring the local cluster's etcd performs well is particularly important since it affects Rancher's ability to manage all downstream clusters.
