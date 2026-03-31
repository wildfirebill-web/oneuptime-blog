# How to Configure D3N libaio Settings for RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, D3N, libaio, RGW, Performance, Async IO

Description: Configure libaio asynchronous I/O settings for D3N in Ceph RGW to improve cache read and write performance with non-blocking disk operations.

---

## Overview

D3N uses libaio (Linux asynchronous I/O) to perform non-blocking cache reads and writes. Tuning libaio settings prevents D3N from becoming a bottleneck when serving many concurrent requests. This guide explains the relevant configuration options and how to set them.

## What is libaio?

libaio is a Linux kernel interface for asynchronous I/O operations. Instead of blocking a thread while waiting for a disk read or write to complete, libaio submits the operation and receives a completion notification later. This allows RGW to handle many simultaneous cache operations without spawning excessive threads.

## Installing libaio

```bash
# RHEL/CentOS/Rocky
dnf install libaio -y

# Ubuntu/Debian
apt-get install libaio1 -y

# Verify installation
ldconfig -p | grep libaio
```

## Key D3N libaio Configuration Options

```bash
# Maximum number of pending async I/O operations per RGW instance
ceph config set client.rgw.myzone rgw_d3n_libaio_aio_threads 8

# Maximum number of libaio events per poll cycle
ceph config set client.rgw.myzone rgw_d3n_libaio_aio_num_events 128
```

In `ceph.conf`:

```ini
[client.rgw.myzone]
d3n_l1_local_datacache_enabled = true
d3n_l1_datacache_persistent_path = /var/lib/ceph/rgw/cache
d3n_l1_datacache_size = 10737418240
rgw_d3n_libaio_aio_threads = 8
rgw_d3n_libaio_aio_num_events = 128
```

## Tuning libaio for Your Workload

For high-concurrency environments (many small objects):

```bash
# Increase threads and events for more parallelism
ceph config set client.rgw.myzone rgw_d3n_libaio_aio_threads 16
ceph config set client.rgw.myzone rgw_d3n_libaio_aio_num_events 256
```

For large object sequential reads:

```bash
# Fewer threads with larger event batches
ceph config set client.rgw.myzone rgw_d3n_libaio_aio_threads 4
ceph config set client.rgw.myzone rgw_d3n_libaio_aio_num_events 64
```

## Checking System libaio Limits

```bash
# Check the maximum AIO requests the kernel supports
cat /proc/sys/fs/aio-max-nr

# Check current AIO in use
cat /proc/sys/fs/aio-nr

# Increase if needed
echo 1048576 > /proc/sys/fs/aio-max-nr

# Make permanent
echo "fs.aio-max-nr = 1048576" >> /etc/sysctl.conf
sysctl -p
```

## Monitoring libaio Performance

```bash
# View D3N performance counters including async I/O stats
ceph daemon rgw.myzone perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
for k, v in data.items():
    if 'd3n' in k.lower() or 'aio' in k.lower():
        print(k, v)
"

# Check for libaio errors in RGW log
journalctl -u ceph-radosgw@rgw.myzone --no-pager | grep -i "aio\|libaio\|async" | tail -30
```

## Summary

Configuring libaio settings for D3N allows Ceph RGW to handle concurrent cache operations efficiently without blocking request threads. Tune the number of AIO threads and events based on your concurrency profile - more threads for many small objects, fewer threads with larger batch sizes for sequential large object reads. Always ensure the kernel's aio-max-nr limit is set high enough to support your configured thread count.
