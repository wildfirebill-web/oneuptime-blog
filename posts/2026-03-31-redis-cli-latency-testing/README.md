# How to Use Redis CLI --latency for Latency Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CLI, Latency, Performance, Testing

Description: Learn how to use redis-cli --latency, --latency-history, and --latency-dist to measure and monitor Redis response latency from your client's perspective.

---

`redis-cli --latency` measures the round-trip time of a simple PING command against your Redis server. It helps you detect network issues, server slowdowns, and performance degradation over time.

## Basic Latency Measurement

```bash
redis-cli --latency
```

Output (updates continuously):

```text
min: 0, max: 1, avg: 0.13 (591 samples)
min: 0, max: 1, avg: 0.14 (1184 samples)
min: 0, max: 2, avg: 0.13 (1779 samples)
```

Values are in milliseconds. Press `Ctrl+C` to stop.

## Latency History

`--latency-history` logs latency samples over time, grouping into 15-second windows by default:

```bash
redis-cli --latency-history
```

Output:

```text
min: 0, max: 1, avg: 0.13 (1487 samples) -- 15.01 seconds range
min: 0, max: 1, avg: 0.12 (1492 samples) -- 15.00 seconds range
min: 0, max: 3, avg: 0.15 (1445 samples) -- 15.01 seconds range
```

Change the window with `-i`:

```bash
redis-cli --latency-history -i 60
```

## Latency Distribution

`--latency-dist` displays a color-coded ASCII histogram of latency distribution:

```bash
redis-cli --latency-dist
```

Output:

```text
 0 ms: ######################################################## (99.82%)
 1 ms: ##                                                        (0.15%)
 2 ms:                                                           (0.02%)
 3 ms:                                                           (0.01%)
```

This shows most commands complete in under 1ms, with rare outliers.

## Connecting to a Remote Server

```bash
redis-cli -h redis.example.com -p 6379 -a password --latency
```

The measured latency includes network round-trip time, so remote results will be higher than localhost.

## Interpreting Results

| Avg Latency | Interpretation |
|-------------|----------------|
| < 1ms | Excellent - local or same datacenter |
| 1-5ms | Good - cross-datacenter |
| 5-20ms | High - investigate network or server load |
| > 20ms | Critical - likely server issue or high load |

## Diagnosing High Latency

If latency spikes appear in `--latency-history`, correlate with:

```bash
# Check slow log for commands exceeding threshold
redis-cli SLOWLOG GET 20

# Check if fork is in progress (RDB save)
redis-cli INFO persistence | grep rdb_bgsave_in_progress

# Check memory
redis-cli INFO memory | grep used_memory_human
```

## Continuous Latency Monitoring with Cron

```bash
#!/bin/bash
# latency-check.sh - run as a cron job every minute
RESULT=$(redis-cli --latency -i 10 2>&1 | tail -1)
echo "$(date): $RESULT" >> /var/log/redis-latency.log
```

Schedule it:

```bash
* * * * * /usr/local/bin/latency-check.sh
```

## Difference Between --latency and --intrinsic-latency

- `--latency`: measures round-trip time over the network (client perspective)
- `--intrinsic-latency`: measures latency caused by the OS on the server itself

Always run `--intrinsic-latency` on the Redis server host, not the client.

## Summary

`redis-cli --latency` measures Redis response latency from the client's perspective using PING commands. Use `--latency-history` to track latency trends over time and `--latency-dist` for distribution visualization. High latency warrants investigation of slow commands, network congestion, or server-side fork operations.
