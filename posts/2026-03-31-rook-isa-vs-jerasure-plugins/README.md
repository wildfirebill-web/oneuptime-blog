# How to Choose Between ISA and Jerasure Plugins for Erasure Coding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, ErasureCoding, Storage, Performance

Description: Learn how to choose between the ISA-L and Jerasure erasure coding plugins in Ceph, and how to configure the right one for your hardware and workload.

---

Ceph supports multiple erasure coding plugins that implement the math behind data protection. The two most common are **Jerasure** (the default) and **ISA-L** (Intel Storage Acceleration Library). Choosing the right plugin can significantly affect encoding and decoding throughput, especially on Intel hardware.

## Understanding the Two Plugins

**Jerasure** is the traditional Ceph erasure coding plugin. It is open source, portable, and works on any x86 or ARM platform. It uses Reed-Solomon or Cauchy-based coding matrices and is the default when you create an erasure code profile without specifying a plugin.

**ISA-L** is Intel's hardware-accelerated library. It uses SIMD instructions (SSE, AVX2, AVX-512) available on modern Intel CPUs to dramatically increase encoding throughput. ISA-L can deliver 2-5x better performance than Jerasure on Intel platforms for large I/O operations.

## Checking Plugin Availability

Before choosing, verify which plugins are installed on your OSD nodes:

```bash
ceph osd erasure-code-profile set test-profile plugin=isa technique=reed_sol_van k=4 m=2
ceph osd erasure-code-profile get test-profile
```

If ISA-L is not installed, the profile creation will fail. On RHEL/CentOS nodes, install it with:

```bash
dnf install isa-l
```

On Ubuntu/Debian:

```bash
apt-get install libisal-dev
```

## Creating an Erasure Code Profile with ISA-L

```bash
ceph osd erasure-code-profile set isa-profile \
  plugin=isa \
  technique=reed_sol_van \
  k=4 \
  m=2
```

To use the default Jerasure plugin:

```bash
ceph osd erasure-code-profile set jerasure-profile \
  plugin=jerasure \
  technique=reed_sol_van \
  k=4 \
  m=2
```

## Performance Comparison

The performance difference is most visible with large sequential writes. Below is a rough benchmark comparison:

```text
Plugin      Encode (GB/s)   Decode (GB/s)   CPU Usage
Jerasure    ~1.2            ~1.5            Higher
ISA-L       ~4.8            ~5.5            Lower (SIMD offload)
```

ISA-L achieves better throughput because the SIMD operations process multiple GF(2^8) field operations in parallel. This reduces CPU time per byte encoded.

## Rook-Ceph Pool Using ISA-L Profile

After defining the profile, create a pool that references it in your Rook CephBlockPool:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-isa-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    erasure_code_profile: isa-profile
```

## When to Use Each Plugin

Use **ISA-L** when:
- All OSD nodes run on modern Intel CPUs (Haswell or newer)
- You need maximum encoding throughput for large objects (RGW workloads)
- CPU budget is a concern and you want to reduce OSD CPU usage

Use **Jerasure** when:
- You have a mixed hardware environment (AMD, ARM, older Intel)
- You need maximum portability and vendor neutrality
- You run on non-Intel cloud instances

## ARM Considerations

On ARM platforms, neither ISA-L nor Jerasure provides SIMD acceleration by default. The `shec` plugin is sometimes preferred on ARM for its simpler computation model, though it offers lower fault tolerance math. For ARM-based Kubernetes clusters running Rook, Jerasure remains the safest default.

## Summary

ISA-L outperforms Jerasure on Intel hardware thanks to vectorized SIMD instructions, making it the preferred choice for high-throughput RGW or archival workloads on Intel OSD nodes. Jerasure remains the portable default for mixed or non-Intel environments. Always verify plugin availability on your nodes before setting a production erasure code profile.
