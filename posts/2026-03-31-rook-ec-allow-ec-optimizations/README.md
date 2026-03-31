# How to Enable Erasure Coding Optimizations (allow_ec_optimizations) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, ErasureCoding, BlueStore, Performance

Description: Learn how to enable allow_ec_optimizations on Ceph erasure coded pools to reduce read-modify-write overhead and improve overwrite performance with BlueStore.

---

Starting with Ceph Quincy (17.x), Ceph introduced the `allow_ec_optimizations` flag for erasure coded pools. This flag enables a smarter write path that reduces unnecessary read-modify-write (RMW) operations, improving performance for workloads that overwrite aligned stripes or append data to EC pools.

## What allow_ec_optimizations Does

When `allow_ec_overwrites` is enabled, every partial write to an EC pool triggers a full RMW cycle - even if the write aligns perfectly to a stripe boundary. The `allow_ec_optimizations` flag adds logic to detect when a write covers a full stripe and bypass the read phase entirely, directly encoding and writing the new data.

This optimization is particularly valuable for:
- RBD sequential workloads that write in stripe-aligned chunks
- RGW multipart uploads with part sizes aligned to the stripe width
- CephFS data pool writes from MDS that issue full-stripe writes

## Requirements

- Ceph Quincy (17.2+) or later
- All OSDs must use BlueStore
- `allow_ec_overwrites` must already be enabled on the pool

## Enabling the Optimization

First, confirm your Ceph version supports the flag:

```bash
ceph version
```

Expected: `ceph version 17.x.x` or higher.

Enable overwrites and then optimizations:

```bash
ceph osd pool set ec-pool allow_ec_overwrites true
ceph osd pool set ec-pool allow_ec_optimizations true
```

Verify both are active:

```bash
ceph osd pool get ec-pool allow_ec_overwrites
ceph osd pool get ec-pool allow_ec_optimizations
```

## Rook CephBlockPool Configuration

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-optimized-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    allow_ec_overwrites: "true"
    allow_ec_optimizations: "true"
```

## Performance Impact

Without optimizations, a full-stripe write to an EC pool (k=4, stripe=256 KiB) still reads 4 chunks before writing. With optimizations enabled, full-stripe writes skip the read:

```text
Write Type              Without Optimization   With Optimization
Full stripe write       Read + Encode + Write  Encode + Write only
Partial stripe write    Read + Encode + Write  Read + Encode + Write
Append (new stripe)     Encode + Write         Encode + Write
```

The savings are most significant for workloads that write in multiples of the logical stripe size. For RBD workloads with a 256 KiB stripe (k=4, stripe_unit=64 KiB), sequential I/O using 256 KiB or larger I/O blocks benefits the most.

## Monitoring RMW Operations

You can verify the optimization is reducing RMW cycles by monitoring OSD perf stats:

```bash
ceph daemon osd.0 perf dump | grep ec_
```

Look for `ec_read_in_progress` counter - it should be lower when most writes hit aligned full-stripe paths.

## Limitations

- The optimization only fires when the write fully covers one or more stripes
- Random sub-stripe writes still incur full RMW overhead
- This is a best-effort optimization; Ceph falls back to RMW when needed

## Summary

`allow_ec_optimizations` reduces read-modify-write overhead for full-stripe overwrites in erasure coded pools. It requires Ceph Quincy or newer and BlueStore OSDs. Enable it alongside `allow_ec_overwrites` for any EC pool used with RBD or CephFS to recover much of the write performance lost to the RMW cycle, especially for sequential workloads.
