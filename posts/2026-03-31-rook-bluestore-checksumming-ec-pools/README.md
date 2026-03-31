# How to Detect Bitrot with BlueStore Checksumming in Erasure Coded Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, BlueStore, ErasureCoding, DataIntegrity

Description: Learn how BlueStore checksumming detects bitrot in Ceph erasure coded pools, how to configure checksum algorithms, and how to trigger scrubbing for proactive detection.

---

Bitrot is the silent corruption of data stored on disk due to hardware faults, firmware bugs, or cosmic rays flipping bits. Ceph's BlueStore OSD backend provides built-in checksumming that detects bitrot at the storage layer, and this is particularly valuable for erasure coded pools where corruption in one shard can be repaired using the remaining shards.

## How BlueStore Checksums Work

BlueStore stores a checksum alongside every block written to the underlying device. When data is read back, BlueStore recomputes the checksum and compares it to the stored value. A mismatch indicates corruption.

For EC pools, if BlueStore detects a corrupted shard during a read:
1. Ceph reads the remaining healthy shards
2. The EC decoder reconstructs the corrupted shard
3. The corrupted OSD is repaired in-place (if scrubbing) or the reconstruction is returned to the client

## Supported Checksum Algorithms

```text
Algorithm    Speed    Strength    Default
crc32c       Fast     Good        Yes
xxhash32     Fast     Moderate    No
xxhash64     Fast     Good        No
sha1         Slow     Strong      No
sha256       Slow     Strong      No
sha512       Slow     Strongest   No
none         Fastest  None        No
```

## Configuring Checksum Algorithm

The default `crc32c` is hardware-accelerated on Intel and ARM CPUs and is the best choice for most deployments. To change it:

```bash
ceph config set osd bluestore_csum_type crc32c
```

Or set per-device class:

```bash
ceph config set osd.0 bluestore_csum_type crc32c
```

To verify the current setting:

```bash
ceph config get osd bluestore_csum_type
```

## Checksum Block Size

BlueStore computes checksums over blocks of data. The `bluestore_csum_block_size` (default: min_alloc_size, usually 4 KiB for HDD or 16 KiB for SSD) controls granularity:

```bash
ceph config get osd bluestore_min_alloc_size_hdd
ceph config get osd bluestore_min_alloc_size_ssd
```

Smaller block sizes detect more localized corruption but increase metadata overhead.

## Triggering Scrub to Find Bitrot

Scrubbing reads all objects and their checksums to proactively detect bitrot before a client requests the corrupted data:

```bash
# Deep scrub a specific pool (reads and verifies all data)
ceph osd pool set ec-pool nodeep-scrub 0
ceph pg deep-scrub 8.0
```

To deep scrub all PGs in a pool:

```bash
for pg in $(ceph pg ls-by-pool ec-pool | awk '{print $1}'); do
  ceph pg deep-scrub $pg
done
```

## Reading Scrub Results

```bash
ceph pg 8.0 query | python3 -c "
import json, sys
data = json.load(sys.stdin)
print('Last scrub:', data.get('info', {}).get('history', {}).get('last_scrub_stamp'))
print('Last deep scrub:', data.get('info', {}).get('history', {}).get('last_deep_scrub_stamp'))
"
```

When bitrot is detected during scrubbing:

```text
[ERR] 2024-01-01T00:00:00.000 osd.3: object ec-pool/oid shard 2 bluestore checksum mismatch
```

Ceph then automatically repairs the shard using the healthy EC shards.

## EC Pool Repair Advantage

Unlike replicated pools where corruption in one copy might be silently returned to clients before scrub detects it, EC pools provide a natural cross-check: if one shard has a checksum mismatch, Ceph can reconstruct the correct data from the remaining k shards and verify it against the corrupted shard.

## Summary

BlueStore checksumming detects bitrot in EC pool shards and triggers automatic repair by reconstructing the corrupted shard from healthy peers. Use `crc32c` (the default) for hardware-accelerated detection, and schedule regular deep scrubs to proactively identify and repair corruption before it affects client reads. EC pools benefit more from checksumming than replicated pools because cross-shard reconstruction provides an additional verification mechanism.
