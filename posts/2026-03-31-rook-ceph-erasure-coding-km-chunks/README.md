# How to Understand Ceph Erasure Coding (K+M Chunks)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, ErasureCoding, Storage, Efficiency, Kubernetes

Description: Understand how Ceph erasure coding splits objects into K data chunks and M coding chunks to provide fault tolerance with lower storage overhead than replication.

---

## What Is Erasure Coding

Erasure coding is a data protection method that divides an object into K data chunks and M coding (parity) chunks, distributing all K+M chunks across separate OSDs. The cluster can reconstruct the original object from any K of the K+M chunks, meaning it can tolerate up to M simultaneous OSD or disk failures.

Erasure coding uses less raw storage than full replication. A 4+2 profile requires 1.5x the original data size (6 chunks total for 4 data chunks), versus 3x for 3-replica.

## How K+M Works

For an object of 4 MB with a 4+2 profile:

- Object is split into 4 data chunks of 1 MB each (K=4)
- 2 coding chunks of 1 MB are computed from the data chunks (M=2)
- All 6 chunks are stored on 6 different OSDs

If OSDs 2 and 5 fail, the remaining 4 chunks (any 4 of 6) are sufficient to reconstruct the original object.

```text
Storage overhead = (K + M) / K = (4 + 2) / 4 = 1.5x
Fault tolerance = M = 2 failures
```

## Creating an Erasure Code Profile in Rook

Define the profile and pool via Rook CRDs:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  failureDomain: host
  parameters:
    compression_mode: none
```

Inspect the resulting profile:

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- bash

# List erasure code profiles
ceph osd erasure-code-profile ls

# Inspect a profile
ceph osd erasure-code-profile get rook-ceph-ec-profile
```

## Common Erasure Code Profiles

| Profile | K | M | Overhead | Fault Tolerance |
|---------|---|---|----------|----------------|
| 2+1 | 2 | 1 | 1.5x | 1 failure |
| 4+2 | 4 | 2 | 1.5x | 2 failures |
| 8+3 | 8 | 3 | 1.375x | 3 failures |
| 4+1 | 4 | 1 | 1.25x | 1 failure |

## Erasure Code Plugin Options

Ceph supports multiple erasure code plugins:

```bash
# Default jerasure plugin (good balance)
ceph osd erasure-code-profile set my-ec k=4 m=2 plugin=jerasure technique=reed_sol_van

# ISA plugin (Intel SIMD acceleration, higher throughput)
ceph osd erasure-code-profile set my-ec-isa k=4 m=2 plugin=isa technique=reed_sol_van

# Clay plugin (repair bandwidth optimization)
ceph osd erasure-code-profile set my-ec-clay k=4 m=2 plugin=clay
```

## Erasure Coded Pool Limitations

Erasure coded pools have important limitations to understand:

- **No partial overwrites**: EC pools cannot do partial in-place overwrites on objects. RBD requires a replicated metadata pool alongside an EC data pool.
- **Higher read amplification**: Reads may require fetching multiple chunks.
- **More OSDs required**: You need at least K+M OSDs (e.g., 6 for 4+2).

For RBD with an EC data pool, configure in Rook as:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-block-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  failureDomain: host
```

## Verify Erasure Coding Is Active

```bash
# Confirm pool is using EC
ceph osd pool get ec-pool erasure_code_profile

# Check chunk distribution
ceph pg dump | grep ec-pool | head -5
```

## Summary

Ceph erasure coding with K data and M coding chunks provides tunable fault tolerance at significantly lower storage cost than replication. A 4+2 profile delivers the same 2-failure tolerance as 3-replica but uses only 1.5x overhead instead of 3x. Understanding K+M parameters, plugin selection, and the limitations of EC pools helps you make the right storage policy decisions in your Rook-Ceph cluster.
