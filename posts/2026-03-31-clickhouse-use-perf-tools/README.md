# How to Use perf Tools with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, perf, Flamegraph, Profiling, Linux, CPU Performance

Description: Use Linux perf tools to profile ClickHouse at the OS level, capture CPU cycles, cache misses, and branch mispredictions to identify native code bottlenecks.

---

## Why Use perf with ClickHouse?

ClickHouse's built-in query profiler samples stack traces at the application level. Linux `perf` provides deeper hardware-level profiling: CPU cycle counts, cache miss rates, branch mispredictions, and instruction-level hotspots. This is useful when ClickHouse's profiler shows a function is slow but you need to know why at the hardware level.

## Prerequisites

Install perf tools:

```bash
apt-get install linux-perf linux-tools-generic
# or
yum install perf
```

Enable kernel perf events:

```bash
sysctl -w kernel.perf_event_paranoid=1
sysctl -w kernel.kptr_restrict=0
```

## Basic CPU Profiling

Find the ClickHouse server PID:

```bash
pgrep -x clickhouse-server
```

Attach perf to the running process:

```bash
perf record -p $(pgrep -x clickhouse-server) -g --call-graph dwarf -F 99 -- sleep 30
```

Generate a report:

```bash
perf report --stdio | head -100
```

## Generating a Flamegraph

Use Brendan Gregg's FlameGraph scripts:

```bash
git clone https://github.com/brendangregg/FlameGraph
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > clickhouse.svg
```

Open `clickhouse.svg` in a browser to explore the interactive flamegraph.

## Profiling a Specific Query

Run perf while executing a known slow query:

```bash
# Start perf in background
perf record -p $(pgrep -x clickhouse-server) -g -F 99 &

# Run the slow query
clickhouse-client --query "
SELECT user_id, sum(revenue) FROM orders
WHERE order_date >= '2026-01-01'
GROUP BY user_id
ORDER BY sum(revenue) DESC
LIMIT 1000
"

# Stop recording
kill %1
perf report --stdio | head -50
```

## Profiling Cache and Branch Behavior

```bash
perf stat -e cache-references,cache-misses,branch-misses,instructions,cycles \
  -p $(pgrep -x clickhouse-server) -- sleep 10
```

Sample output:

```text
  1,234,567,890  cache-references
    123,456,789  cache-misses       # 10.0% of all cache refs
      5,678,901  branch-misses
 45,678,901,234  instructions
 23,456,789,012  cycles
```

High cache miss rates with ClickHouse often point to random-access patterns in JOINs or dictionaries.

## Reading ClickHouse Debug Symbols

For meaningful function names in perf output, install debug symbols:

```bash
apt-get install clickhouse-common-static-dbg
```

Or build ClickHouse from source with debug symbols enabled.

## Summary

Use Linux `perf` with ClickHouse by attaching to the server process with `perf record -g`, generating flamegraphs with Brendan Gregg's scripts, and using `perf stat` for hardware counter profiling. Install debug symbols for readable function names. Combine `perf` output with ClickHouse's own trace log for a complete performance picture.
