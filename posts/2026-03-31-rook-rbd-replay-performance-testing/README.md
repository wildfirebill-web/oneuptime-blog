# How to Use RBD Replay for Performance Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Performance, Testing

Description: Learn how to use RBD replay to capture and replay I/O workloads against Ceph block devices for performance testing and regression analysis.

---

## What Is RBD Replay

RBD replay is a two-phase tool for capturing and replaying I/O workloads against RBD block devices. It consists of:

- `rbd-nbd` with `--capture` or using `blktrace` to record I/O operations
- `rbd replay` to replay the captured I/O against a target RBD image

This allows you to benchmark Ceph cluster performance using realistic workloads captured from production systems, and to compare performance before and after configuration changes.

## Step 1 - Prerequisites

Ensure the following are available in the Rook toolbox or on the test node:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- bash
rbd --version
which blktrace
```

Install blktrace if not present:

```bash
apt-get install -y blktrace
```

## Step 2 - Map an RBD Image for Capture

Map the source RBD image to a block device on the test node:

```bash
rbd map replicapool/myimage
```

Note the device name (e.g., `/dev/rbd0`).

## Step 3 - Capture I/O with rbd-nbd and blktrace

Start capturing I/O from the block device using blktrace:

```bash
blktrace -d /dev/rbd0 -o /tmp/io-capture -w 60
```

This captures 60 seconds of I/O operations. The output files will be `/tmp/io-capture.blktrace.0` etc.

Convert the capture to a format usable by rbd-replay:

```bash
blkparse -i /tmp/io-capture -d /tmp/io-capture.bin -O
rbd-replay-prep /tmp/io-capture.bin /tmp/rbd-replay-workload
```

## Step 4 - Set Up a Target Image for Replay

Create a target image for replay (same size as source):

```bash
rbd create replicapool/replay-target --size 10G
rbd map replicapool/replay-target
```

## Step 5 - Run the Replay

Replay the captured workload against the target image:

```bash
rbd-replay --map-whole-image --latency-multiplier=1 \
  /tmp/rbd-replay-workload /dev/rbd1
```

Key flags:
- `--latency-multiplier` - Scale replay speed (1 = original speed, 0 = as fast as possible)
- `--map-whole-image` - Map all I/O to the target image regardless of original offsets

## Step 6 - Measure Throughput During Replay

While replay runs, measure I/O performance using iostat:

```bash
iostat -x /dev/rbd1 1
```

Sample output during replay:

```text
Device  r/s    w/s   rkB/s    wkB/s  await  util
rbd1    120.5  85.3  3820.4   2732.8  2.1    45.2
```

## Step 7 - Compare Before and After

Run the same replay before and after tuning librbd settings, changing CRUSH rules, or adding OSDs:

```bash
# Before tuning
rbd-replay --latency-multiplier=0 /tmp/rbd-replay-workload /dev/rbd1 \
  2>&1 | tee /tmp/before-tuning.log

# After tuning - run again with same capture
rbd-replay --latency-multiplier=0 /tmp/rbd-replay-workload /dev/rbd1 \
  2>&1 | tee /tmp/after-tuning.log
```

Compare total I/O time and latency distributions between both runs.

## Summary

RBD replay provides a rigorous method for performance testing Ceph block storage by capturing real workloads with blktrace and replaying them against test images. The workflow involves capturing I/O from production, converting to rbd-replay format, and running the replay with `--latency-multiplier=0` for maximum speed benchmarking. This enables objective before-and-after comparisons when tuning Rook-Ceph configurations.
