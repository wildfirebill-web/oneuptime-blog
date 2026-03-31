# How to Optimize Ceph Read Performance for Data Loading

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Performance, Data Loading, Machine Learning

Description: Optimize Ceph read performance for ML data loading pipelines by tuning client caches, OSD parameters, and network settings for maximum throughput.

---

## Overview

ML data loading is often the hidden bottleneck in training pipelines. PyTorch DataLoader workers, TensorFlow tf.data pipelines, and custom loaders all make parallel read requests to storage. Ceph must be tuned to sustain high concurrency without latency spikes.

## Profile Current Data Loading Performance

Measure data loading throughput before tuning:

```python
import time
import torch
from torch.utils.data import DataLoader, Dataset

class CephDataset(Dataset):
    def __init__(self, path):
        import glob
        self.files = glob.glob(f"{path}/*.pt")

    def __len__(self):
        return len(self.files)

    def __getitem__(self, idx):
        return torch.load(self.files[idx])

dataset = CephDataset("/mnt/cephfs/training-data")
loader = DataLoader(dataset, batch_size=32, num_workers=8, pin_memory=True)

start = time.time()
for i, batch in enumerate(loader):
    if i == 100:
        break
elapsed = time.time() - start
print(f"Throughput: {100 * 32 / elapsed:.1f} samples/sec")
```

## Tune Kernel Page Cache for CephFS

The Linux kernel page cache helps with repeated reads:

```bash
# On each Kubernetes node, increase vm.dirty settings for write-through caching
cat /proc/sys/vm/dirty_ratio
echo 10 > /proc/sys/vm/dirty_ratio
echo 5 > /proc/sys/vm/dirty_background_ratio
```

## Increase DataLoader Workers Safely

More workers cause more parallel Ceph reads:

```python
import multiprocessing

# Use num_workers = 2x available CPUs for I/O-bound loading
num_workers = min(multiprocessing.cpu_count() * 2, 16)

loader = DataLoader(
    dataset,
    batch_size=64,
    num_workers=num_workers,
    prefetch_factor=4,
    persistent_workers=True
)
```

## Tune Ceph Client for Parallel Reads

```bash
# Increase objecter parallelism
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set client objecter_inflight_ops 24576

# Larger RBD read-ahead
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set client rbd_readahead_max_bytes 33554432

# Enable RBD cache for block volumes
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set client rbd_cache true

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set client rbd_cache_size 268435456
```

## Pre-shuffle Datasets on Ceph

Rather than shuffling during loading (causing random I/O), pre-shuffle and store:

```python
import random
import os

# Pre-shuffle file list and store mapping
files = sorted(os.listdir('/mnt/cephfs/training-data'))
random.shuffle(files)

with open('/mnt/cephfs/training-data/shuffle-index.txt', 'w') as f:
    f.write('\n'.join(files))
```

## Monitor Read Performance

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool stats ml-datasets

kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph iostat 5
```

## Summary

Optimizing Ceph read performance for data loading involves tuning client-side concurrency parameters, enabling RBD cache for block volumes, adjusting kernel page cache settings, and using persistent DataLoader workers with prefetching. Pre-shuffling datasets eliminates random I/O patterns, allowing Ceph to serve sequential reads at full throughput.
