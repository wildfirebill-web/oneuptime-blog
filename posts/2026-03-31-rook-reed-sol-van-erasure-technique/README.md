# How to Select Erasure Coding Techniques (reed_sol_van) in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, ErasureCoding, Storage, ReedSolomon

Description: Learn how to select the right erasure coding technique in Ceph, including reed_sol_van, cauchy_good, and others, and when each technique is appropriate.

---

Ceph's erasure coding supports multiple mathematical techniques for constructing coding matrices. The technique controls how data chunks and coding chunks are computed from each other. Selecting the correct technique affects both performance and recovery capabilities.

## Available Techniques

The Jerasure plugin supports the following techniques:

```text
reed_sol_van       - Vandermonde Reed-Solomon (default, most common)
reed_sol_r6_op     - Optimized for m=2 (RAID-6 equivalent)
cauchy_orig        - Original Cauchy matrix coding
cauchy_good        - Improved Cauchy matrix, less XOR operations
liberation         - XOR-based, efficient for m=2
blaum_roth         - XOR-based with different stripe properties
liber8tion         - Optimized XOR technique for m=2
```

The ISA plugin supports:

```text
reed_sol_van       - Vandermonde matrix (same math, hardware accelerated)
cauchy              - Cauchy matrix encoding
```

## Setting a Technique in a Profile

```bash
ceph osd erasure-code-profile set my-rs-profile \
  plugin=jerasure \
  technique=reed_sol_van \
  k=4 \
  m=2
```

To use the RAID-6 optimized technique (only valid with m=2):

```bash
ceph osd erasure-code-profile set my-r6-profile \
  plugin=jerasure \
  technique=reed_sol_r6_op \
  k=6 \
  m=2
```

## reed_sol_van: The Standard Choice

`reed_sol_van` uses a Vandermonde matrix over GF(2^w) (default w=8). It is the most widely tested technique and supports any combination of k and m. It is recommended for production use when you do not have specific performance constraints requiring a specialized technique.

The math ensures that any m chunks (data or coding) can be lost and the original data fully recovered, as long as at least k chunks remain available.

## Cauchy vs Vandermonde

Cauchy matrices can outperform Vandermonde in encoding speed for some k/m combinations because they require fewer XOR operations per bit. The tradeoff is slightly higher matrix construction overhead:

```text
Technique       k=4,m=2 XORs   Portability   Notes
reed_sol_van    High            Excellent     Default
cauchy_good     Medium          Good          Fewer XORs per word
reed_sol_r6_op  Low             m=2 only      RAID-6 optimized
```

## XOR-Based Techniques for m=2

If you only need protection against 2 simultaneous failures (m=2), techniques like `liberation`, `blaum_roth`, and `liber8tion` use pure XOR operations instead of GF field multiplications. This reduces CPU overhead significantly:

```bash
ceph osd erasure-code-profile set xor-profile \
  plugin=jerasure \
  technique=liber8tion \
  k=6 \
  m=2
```

These XOR-based techniques are limited to m=2 and specific stripe width constraints.

## Verifying a Profile

```bash
ceph osd erasure-code-profile get my-rs-profile
```

Expected output:

```text
crush_failure_domain=host
crush_root=default
k=4
m=2
plugin=jerasure
technique=reed_sol_van
```

## Rook Integration

When using Rook, you first create the profile via the Ceph toolbox, then reference it in a CephBlockPool:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-rs-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    erasure_code_profile: my-rs-profile
```

## Summary

`reed_sol_van` is the default and most versatile erasure coding technique in Ceph, suitable for any k/m combination in production. For RAID-6 equivalent setups (m=2), `reed_sol_r6_op` or XOR-based techniques like `liber8tion` offer better encoding performance. Choose your technique based on the value of m and the CPU budget available on OSD nodes.
