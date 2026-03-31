# How to Perform CPU Profiling in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CPU Profiling, Performance, Debugging

Description: Learn how to perform CPU profiling on Ceph daemons using gperftools CPU profiler and perf to identify hotspots and optimize daemon performance.

---

CPU profiling Ceph daemons helps identify which code paths consume the most processor time, enabling targeted optimization of OSD, monitor, and MDS performance. Ceph supports CPU profiling through the gperftools CPU profiler and Linux perf.

## CPU Profiling with the Admin Socket

Ceph daemons expose CPU profiler start/stop commands via the admin socket when built with gperftools:

```bash
# Start CPU profiling on OSD 0
ceph tell osd.0 cpu_profiler start

# Run your workload here
fio --filename=/dev/rbd0 --readwrite=randrw --bs=4k --numjobs=4 --runtime=30 --name=test

# Stop profiling and write the profile
ceph tell osd.0 cpu_profiler stop

# Status check
ceph tell osd.0 cpu_profiler status
```

Profile files are written to `/tmp/ceph-osd.0.cpuprofile`.

## Analyzing CPU Profiles with pprof

```bash
# Text-based report showing top functions
google-pprof --text /usr/bin/ceph-osd /tmp/ceph-osd.0.cpuprofile | head -30

# Generate a flame graph-compatible output
google-pprof --collapse /usr/bin/ceph-osd /tmp/ceph-osd.0.cpuprofile

# Generate a call graph PDF
google-pprof --pdf /usr/bin/ceph-osd /tmp/ceph-osd.0.cpuprofile > cpu_profile.pdf
```

Example pprof text output:

```yaml
Total: 1500 samples
     300  20.0%  20.0%      300  20.0%  BlueStore::_do_write
     180  12.0%  32.0%      180  12.0%  PG::do_osd_ops
     120   8.0%  40.0%      450  30.0%  OSD::dispatch_op
```

## Linux perf for System-Level Profiling

For broader CPU analysis including kernel space:

```bash
# Find the OSD process ID
PID=$(pgrep -f "ceph-osd -i 0")

# Profile for 30 seconds
sudo perf record -g -p $PID -o osd_perf.data -- sleep 30

# Generate a report
sudo perf report -i osd_perf.data

# Generate flamegraph data
sudo perf script -i osd_perf.data | \
  stackcollapse-perf.pl | \
  flamegraph.pl > osd_flamegraph.svg
```

## Identifying CPU Hotspots via Perf Counters

Use Ceph's built-in performance counters to find high-CPU operations:

```bash
# Watch OSD op processing rates
ceph tell osd.0 perf dump | python3 -m json.tool | grep -A2 "op_r_prepare_lat"

# Monitor all daemons via Prometheus
curl -s http://192.168.1.10:9283/metrics | grep "ceph_osd_op_r_"
```

## Profiling with strace for Syscall Analysis

Identify system call overhead:

```bash
PID=$(pgrep -f "ceph-osd -i 0")
sudo strace -p $PID -c -e trace=io,ipc 2>&1 | head -30
```

## Reducing CPU Load

Common tuning options after identifying hotspots:

```bash
# Reduce unnecessary deep scrubbing
ceph config set osd osd_deep_scrub_interval 604800  # 7 days

# Tune recovery thread count
ceph config set osd osd_recovery_threads 1

# Disable expensive debug logging
ceph config set osd debug_osd 0
ceph config set osd debug_bluestore 0
```

## Summary

Ceph CPU profiling uses gperftools through `ceph tell osd.X cpu_profiler` commands to capture call-count profiles, analyzed with `google-pprof` to identify top CPU-consuming functions. Linux `perf record` complements this with system-level profiling including kernel callstacks, and flame graphs provide the most intuitive visual representation of CPU hotspots across the daemon's execution paths.
