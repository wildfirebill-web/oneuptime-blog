# How to Use Redis CLI --intrinsic-latency for System Latency

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CLI, Latency, System, Performance

Description: Learn how to use redis-cli --intrinsic-latency to measure OS-induced latency on your Redis server host, helping you tune system settings for low-latency workloads.

---

Intrinsic latency is the delay introduced by the operating system itself - from scheduler jitter, memory page faults, NUMA effects, and background processes. Unlike network latency, intrinsic latency exists even on localhost. Redis performance is bounded by this floor.

## What --intrinsic-latency Measures

`--intrinsic-latency` runs a tight loop on the server host for a specified duration and measures the worst-case delay it experiences from the OS. This tells you the minimum latency Redis can achieve on that machine.

## Running --intrinsic-latency

**Run this on the Redis server host itself**, not from a client machine:

```bash
redis-cli --intrinsic-latency 30
```

The argument is the number of seconds to run the test. Longer durations give more accurate worst-case measurements.

Output:

```text
Max latency so far: 1 microseconds.
Max latency so far: 7 microseconds.
Max latency so far: 14 microseconds.
Max latency so far: 28 microseconds.
Max latency so far: 83 microseconds.

3052099 total runs (avg latency: 0.9827 microseconds / 982.70 nanoseconds per run).
Worst run took 85x longer than the average latency.
```

A "worst run" 10x or more higher than average indicates the OS is introducing jitter.

## Interpreting Results

| Max Latency | System Quality |
|-------------|----------------|
| < 100 microseconds | Excellent |
| 100 - 500 microseconds | Acceptable for most workloads |
| 500 - 1000 microseconds | Needs tuning |
| > 1 ms | Significant OS interference |

## Tuning the OS to Reduce Intrinsic Latency

### Disable Transparent Huge Pages

THP causes latency spikes during memory compaction:

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

Make permanent in `/etc/rc.local`.

### Disable NUMA Balancing

```bash
echo 0 > /proc/sys/kernel/numa_balancing
```

### Increase CPU Governor to Performance Mode

```bash
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
  echo performance > $cpu
done
```

### Disable Swap

```bash
swapoff -a
```

### Pin Redis to CPUs

Use `taskset` to pin Redis to specific CPU cores:

```bash
taskset -c 0,1 redis-server /etc/redis/redis.conf
```

## Re-Testing After Tuning

```bash
redis-cli --intrinsic-latency 60
```

Compare the new worst-case latency to the baseline.

## Comparing Before and After

```text
Before THP disabled:
  Max latency: 2400 microseconds
  Worst run: 2400x average

After THP disabled:
  Max latency: 85 microseconds
  Worst run: 85x average
```

## When to Run This Test

Run `--intrinsic-latency` when:
- Setting up a new Redis server
- Investigating unexplained latency spikes
- After OS upgrades
- When moving to new hardware or VM types

## Summary

`redis-cli --intrinsic-latency` measures OS-induced jitter on the Redis server host, setting a floor on the latency Redis can achieve. High intrinsic latency points to THP, NUMA balancing, CPU throttling, or swap activity. Addressing these at the OS level directly improves Redis response times.
