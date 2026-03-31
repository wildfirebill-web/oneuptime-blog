# How to Profile Redis Network Latency

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Latency, Network, Performance, Profiling

Description: Learn how to measure and diagnose Redis network latency using built-in tools, ping tests, and network analysis to isolate where latency comes from.

---

When Redis is slow, the first step is isolating whether the latency is in Redis itself, the network, or client-side. Redis provides several built-in tools for network latency measurement.

## Baseline Latency with redis-cli --latency

```bash
redis-cli -h redis-server-ip -p 6379 --latency
```

Output (samples every 100ms):

```text
min: 0, max: 1, avg: 0.43 (1200 samples)
```

Run for a longer period to catch spikes:

```bash
redis-cli -h redis-server-ip -p 6379 --latency-history -i 5
```

This shows a new sample every 5 seconds, revealing patterns over time.

## Measure Intrinsic Latency

Measure baseline OS latency (not Redis-specific):

```bash
redis-cli --intrinsic-latency 30
```

Run this on the Redis server itself. It measures the maximum time the OS delays a process. Any Redis latency close to this number is OS-imposed, not Redis-imposed.

```text
Max latency so far: 1 microseconds
Max latency so far: 16 microseconds
Max latency so far: 78 microseconds

8018485 total runs (avg latency: 3.7414 microseconds / 3741 nanoseconds per run).
Worst run took 20x longer than the average latency.
```

## Compare Local vs Remote Latency

Measure from the Redis server itself:

```bash
redis-cli -h 127.0.0.1 --latency
# min: 0, max: 1, avg: 0.12
```

Measure from application server:

```bash
redis-cli -h 10.0.1.5 --latency
# min: 0, max: 3, avg: 0.85
```

The difference (0.73ms here) is pure network round-trip time.

## Use DEBUG SLEEP to Isolate Redis vs Network

```bash
# Ask Redis to sleep for 100ms, measure total round-trip
time redis-cli DEBUG SLEEP 0.1
```

If this takes significantly more than 100ms, either network or OS scheduling is adding overhead.

## Capture Network Packets for Analysis

```bash
sudo tcpdump -i eth0 -n port 6379 -w /tmp/redis-capture.pcap &
redis-cli -h redis-host SET testkey testval
redis-cli -h redis-host GET testkey
kill %1

# Analyze in Wireshark or tshark
tshark -r /tmp/redis-capture.pcap -q -z "io,stat,0.001,tcp.port==6379"
```

## Check for Network Packet Loss

```bash
ping -c 100 redis-server-ip | tail -5
```

Even 0.1% packet loss causes retransmissions that increase P99 latency significantly.

## Summary

Profile Redis network latency using `redis-cli --latency` for real-time measurements and `--latency-history` for trends. Use `--intrinsic-latency` on the server to establish OS baseline overhead. Compare local vs remote measurements to isolate network contribution. Packet loss even at low rates significantly elevates P99 latency.
