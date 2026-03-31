# How to Understand Ceph BlueStore Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, BlueStore, Architecture, Storage, OSD, Performance, Internals

Description: A deep dive into Ceph BlueStore's architecture, explaining how it directly manages block devices to eliminate double-write overhead.

---

## Overview

BlueStore is Ceph's default OSD backend, introduced in Ceph Luminous (12.x) and replacing FileStore. Unlike FileStore, which stores objects as files on a local filesystem, BlueStore writes directly to raw block devices using its own storage management layer. This eliminates the "double-write penalty" of FileStore and improves performance significantly.

## Why BlueStore Replaced FileStore

FileStore's architecture had fundamental limitations:

- **Double-write penalty** - Data written to journal first, then to the filesystem
- **Filesystem overhead** - ext4 or XFS metadata operations added latency
- **No native checksumming** - Data integrity relied on the filesystem layer
- **Slow EC overwrites** - Partial object updates were expensive

BlueStore addresses all of these.

## BlueStore's Core Components

BlueStore consists of four main components:

```text
+------------------+
|   BlueStore OSD  |
+--------+---------+
         |
+--------v---------+
| Object Metadata  |  <-- RocksDB (stored on DB device or main)
+------------------+
| Data Blocks      |  <-- Direct block device (main device)
+------------------+
| WAL Journal      |  <-- RocksDB WAL (stored on WAL device or main)
+------------------+
```

### 1. RocksDB for Metadata

All object metadata, including object names, xattrs, and omap data, is stored in an embedded RocksDB instance. RocksDB provides atomic transactions and compaction.

```bash
# View RocksDB stats on an OSD
ceph daemon osd.0 perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
for k, v in data.get('rocksdb', {}).items():
    print(k, ':', v)
"
```

### 2. Direct Block Device for Object Data

Object data is written directly to the raw block device in fixed-size allocation units called `min_alloc_size`. This bypasses the kernel filesystem entirely.

```bash
# Check allocation size for an OSD
ceph config show osd.0 bluestore_min_alloc_size_hdd
ceph config show osd.0 bluestore_min_alloc_size_ssd
```

### 3. WAL (Write-Ahead Log)

Small writes that are smaller than `min_alloc_size` are first written to the WAL (stored in RocksDB by default). This ensures atomicity without fragmenting the data device.

### 4. BlueFS

BlueStore uses BlueFS, a minimal log-structured filesystem, solely to host the RocksDB metadata. BlueFS is not a general-purpose filesystem.

## Data Write Path in BlueStore

```text
Client Write Request
        |
        v
+------------------+
| Write >= min_alloc_size?  |
| YES: Write directly to block device |
| NO: Defer to RocksDB WAL  |
+------------------+
        |
        v
+------------------+
| Checksum computed inline |
| Metadata update in RocksDB |
+------------------+
        |
        v
+------------------+
| Acknowledge write to client |
+------------------+
```

## Checksumming in BlueStore

BlueStore computes checksums for all data at write time. Default checksum algorithm is crc32c:

```bash
# View checksum settings
ceph config show osd.0 bluestore_csum_type

# Change checksum algorithm (xxhash64 is faster on some CPUs)
ceph config set osd bluestore_csum_type xxhash64
```

## Compression in BlueStore

BlueStore supports transparent compression:

```bash
# Enable snappy compression on a pool
ceph osd pool set mypool compression_mode aggressive
ceph osd pool set mypool compression_algorithm snappy
ceph osd pool set mypool compression_min_blob_size 8192
```

## Viewing BlueStore Internal State

```bash
# Inspect a specific OSD's BlueStore state
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-0/block list

# Check BlueStore space usage
ceph daemon osd.0 bluestore allocator dump block
```

## Summary

BlueStore's architecture eliminates FileStore's double-write penalty by writing object data directly to raw block devices while using RocksDB for metadata management. Its built-in checksumming, transparent compression, and support for separate WAL and DB devices make it significantly faster than FileStore. Understanding the relationship between the block device, RocksDB, BlueFS, and the WAL is foundational to tuning BlueStore performance for specific workloads.
