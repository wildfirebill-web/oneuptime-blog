# How to Debug Slow Requests via Admin Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Admin Socket, Slow Request, Debug, Latency

Description: Debug slow request warnings in Ceph OSD daemons using admin socket commands to identify stuck operations, blocked PGs, and latency bottlenecks in real time.

---

## Overview

Ceph logs warnings when OSD operations take longer than a threshold (default 30 seconds). These "slow request" warnings often point to resource contention, network issues, or blocked PGs. The admin socket provides commands to inspect in-flight operations and identify what is causing the slowness.

## Viewing In-Flight Operations

```bash
# Show all currently in-flight operations on an OSD
ceph daemon osd.0 dump_ops_in_flight

# Pretty print with Python
ceph daemon osd.0 dump_ops_in_flight | python3 -m json.tool | head -100
```

## Identifying Slow Operations

```bash
# Show only operations that have been waiting longer than 30 seconds
ceph daemon osd.0 dump_ops_in_flight | python3 -c "
import sys, json, time
data = json.load(sys.stdin)
ops = data.get('ops', [])
now = time.time()
slow = []
for op in ops:
    initiated = op.get('initiated_at', 0)
    if isinstance(initiated, str):
        import datetime
        # Parse the timestamp
        pass
    desc = op.get('description', '')
    age = op.get('age', 0)
    if age > 30:
        slow.append((age, desc))
slow.sort(reverse=True)
print(f'Total in-flight: {len(ops)}')
print(f'Slow (>30s): {len(slow)}')
for age, desc in slow[:10]:
    print(f'  {age:.1f}s: {desc}')
"
```

## Using dump_historic_ops

After slow requests complete, they are moved to the historic ops log:

```bash
# View recently completed slow operations
ceph daemon osd.0 dump_historic_ops | python3 -m json.tool

# Show the slowest 20 operations
ceph daemon osd.0 dump_historic_ops | python3 -c "
import sys, json
data = json.load(sys.stdin)
ops = data.get('ops', [])
ops_with_dur = [(op.get('duration', 0), op.get('description', '')) for op in ops]
ops_with_dur.sort(reverse=True)
for dur, desc in ops_with_dur[:20]:
    print(f'{dur:.3f}s: {desc}')
"
```

## Analyzing Operation Stages

Each operation has stages showing where time was spent:

```bash
ceph daemon osd.0 dump_historic_ops | python3 -c "
import sys, json
data = json.load(sys.stdin)
ops = data.get('ops', [])
for op in ops[:5]:
    print('Op:', op.get('description', ''))
    print('Duration:', op.get('duration', 0), 'seconds')
    stages = op.get('type_data', {}).get('events', [])
    for stage in stages:
        print(f'  {stage}')
    print()
"
```

## Common Slow Request Causes

```bash
# 1. Check for bluestore latency issues
ceph daemon osd.0 perf dump | python3 -m json.tool | grep -E "commit_lat|apply_lat"

# 2. Check for recovery/backfill blocking client I/O
ceph daemon osd.0 config get osd_recovery_max_active
ceph daemon osd.0 config get osd_max_backfills

# 3. Check for journal contention (FileStore only)
ceph daemon osd.0 config get journal_write_header_frequency

# 4. Check network messenger latency
ceph daemon osd.0 perf dump | python3 -m json.tool | grep -E "ms.*lat"
```

## Adjusting Slow Request Threshold

```bash
# Default is 30 seconds - adjust for your environment
ceph daemon osd.0 config set osd_op_complaint_time 60

# Or globally
ceph config set osd osd_op_complaint_time 60
```

## Watching for New Slow Requests

```bash
# Monitor OSD log for slow request warnings
journalctl -u ceph-osd@0 -f | grep -E "slow request|blocked|delay"

# Check current slow request count
ceph daemon osd.0 perf dump | python3 -m json.tool | grep "op_latency\|slow_requests"
```

## Summary

The admin socket `dump_ops_in_flight` and `dump_historic_ops` commands are the primary tools for debugging Ceph slow requests. In-flight ops show what is currently stuck, while historic ops reveal the stages where time was spent in recently completed slow operations. Common causes include BlueStore commit latency, recovery I/O interference, and network issues - use the perf counters and config get commands to narrow down the root cause.
