# How to Select Erasure Coding Techniques in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Erasure Coding, Reed Solomon, Storage

Description: Compare Ceph erasure coding techniques including reed_sol_van, reed_sol_r6_op, and cauchy variants to select the right algorithm for your workload and hardware.

---

## Erasure Coding Plugins and Techniques

Ceph supports multiple erasure coding plugins and techniques through the `jerasure` and `isa` libraries. The choice affects encoding/decoding CPU performance, compatibility, and the range of supported K and M values.

```bash
# View available plugins
ceph osd erasure-code-profile ls

# Create a profile specifying technique
ceph osd erasure-code-profile set my-profile \
  k=4 m=2 \
  plugin=jerasure \
  technique=reed_sol_van
```

## Available Techniques

### reed_sol_van (Reed-Solomon Vandermonde)

The default and most widely used technique in Ceph. It uses Vandermonde matrices for encoding, providing excellent compatibility across all K and M combinations.

```bash
ceph osd erasure-code-profile set ec-reedsolvan \
  k=4 m=2 \
  plugin=jerasure \
  technique=reed_sol_van
```

**Characteristics:**
- Supports any combination of K and M
- Good CPU performance on modern hardware
- Most widely tested and deployed in Ceph
- Recommended for general use

### reed_sol_r6_op

Optimized specifically for RAID-6 configurations (m=2 only). Uses a different algorithm that is faster than `reed_sol_van` for 2-parity configurations.

```bash
ceph osd erasure-code-profile set ec-r6 \
  k=4 m=2 \
  plugin=jerasure \
  technique=reed_sol_r6_op
```

**Characteristics:**
- Only supports m=2
- Faster than reed_sol_van for m=2 profiles
- Good choice for standard 2-parity configurations

### cauchy_orig

Original Cauchy matrix-based encoding. Less efficient than `cauchy_good` and generally not recommended.

### cauchy_good

Optimized Cauchy matrix encoding with better performance than `cauchy_orig` by selecting coefficient matrices that minimize XOR operations.

```bash
ceph osd erasure-code-profile set ec-cauchy \
  k=6 m=3 \
  plugin=jerasure \
  technique=cauchy_good
```

**Characteristics:**
- Better performance for larger M values (m=3 or more)
- Slightly lower fault tolerance guarantees than Reed-Solomon (rare edge cases)

### liberation, blaum_roth, liber8tion

RAID-6 optimized techniques. These minimize XOR operations and work best with m=2:

```bash
ceph osd erasure-code-profile set ec-lib \
  k=6 m=2 \
  plugin=jerasure \
  technique=liberation
```

## ISA Plugin (Intel Storage Acceleration)

On Intel hardware with SSE4.2 or AVX-512 support, the `isa` plugin provides hardware-accelerated erasure coding:

```bash
ceph osd erasure-code-profile set ec-isa \
  k=4 m=2 \
  plugin=isa \
  technique=reed_sol_van
```

**Characteristics:**
- Significantly faster on Intel CPUs with AVX-512
- Uses Intel ISA-L library for hardware acceleration
- Same techniques as jerasure but hardware-optimized

## Technique Selection Guide

```text
Use Case                          | Recommended Technique
General purpose (any M)           | reed_sol_van (jerasure)
Standard 2-parity (m=2)           | reed_sol_r6_op or liberation
Performance on Intel hardware     | reed_sol_van (isa plugin)
3+ parity chunks                  | cauchy_good or reed_sol_van
Maximum compatibility             | reed_sol_van
```

## Testing Technique Performance

Use the erasure code tool to benchmark techniques:

```bash
# Test encoding performance
ceph osd erasure-code-profile set test-profile k=4 m=2 plugin=jerasure technique=reed_sol_van

# Encode/decode performance test (requires ceph source tools)
ceph_erasure_code_benchmark \
  --plugin jerasure \
  --parameter technique=reed_sol_van \
  --parameter k=4 \
  --parameter m=2 \
  --iterations 1000
```

## Viewing Profile Details

```bash
# List all profiles
ceph osd erasure-code-profile ls

# View a specific profile
ceph osd erasure-code-profile get my-profile

# Apply profile to a pool
ceph osd pool create mypool 128 128 erasure my-profile
```

## Summary

Ceph offers multiple erasure coding techniques through the jerasure and isa plugins. `reed_sol_van` is the default and most compatible choice for general use. `reed_sol_r6_op` and `liberation` are faster alternatives for standard 2-parity (m=2) configurations. On Intel hardware with AVX-512, the `isa` plugin provides hardware-accelerated encoding. For most production deployments, `reed_sol_van` with the `jerasure` plugin (or `isa` on Intel) is the recommended starting point.
