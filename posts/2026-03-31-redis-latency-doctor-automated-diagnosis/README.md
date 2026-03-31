# How to Use LATENCY DOCTOR in Redis for Automated Diagnosis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, LATENCY DOCTOR, Latency, Performance, Diagnostic

Description: Learn how to use LATENCY DOCTOR in Redis to get an automated human-readable analysis of latency events and actionable tuning recommendations.

---

`LATENCY DOCTOR` analyzes the latency events recorded by Redis and generates a human-readable report with explanations and actionable recommendations. It is the easiest entry point for diagnosing latency spikes in Redis without deep expertise in Redis internals.

## Syntax

```text
LATENCY DOCTOR
```

Returns a formatted text report. Requires `latency-monitor-threshold` to be configured.

## Enabling Latency Monitoring

Before `LATENCY DOCTOR` can provide useful output, enable the latency monitor:

```bash
# Set threshold to 100ms - events slower than this are recorded
redis-cli CONFIG SET latency-monitor-threshold 100

# Alternatively in redis.conf:
# latency-monitor-threshold 100
```

## Running LATENCY DOCTOR

```bash
redis-cli LATENCY DOCTOR
```

If no latency events have been recorded, the output is:

```text
I have no memory of any event. Tom Hanks would be proud of me.
```

With recorded events, you get a detailed report like:

```text
Dave, I have observed latency spikes in the event 'aof-stat'.
Here is a suggestion:
- Rewrite the AOF if it has grown much larger than the actual data set.
I have observed latency spikes in the event 'fast-command'.
Here is a suggestion:
- Check for slow commands using SLOWLOG GET.
```

## Common Latency Events and Their Causes

| Event | Common Cause |
|-------|-------------|
| `aof-stat` | AOF fsync taking too long - often due to slow disk |
| `fast-command` | Slow commands blocking the event loop |
| `rdb-save-in-progress` | Background RDB save impacting performance |
| `expire-cycle` | Key expiration scan taking too long |
| `loading-key-space` | Loading RDB from disk at startup |

## Combining with LATENCY HISTORY

Get more detail by checking history for a specific event:

```bash
# See all recorded occurrences of aof-stat latency
redis-cli LATENCY HISTORY aof-stat
```

## Combining with LATENCY LATEST

See the most recent latency reading for each event:

```bash
redis-cli LATENCY LATEST
```

## Responding to DOCTOR Recommendations

Example remediation for common recommendations:

```bash
# High AOF latency - check and rewrite AOF
redis-cli BGREWRITEAOF

# Slow commands detected - check slowlog
redis-cli SLOWLOG GET 10

# Too many expired keys causing expiry cycle latency
redis-cli CONFIG SET hz 20  # Increase expiry scan frequency
```

## Resetting Latency History

After addressing issues, reset latency history to start fresh:

```bash
redis-cli LATENCY RESET
```

## Summary

`LATENCY DOCTOR` provides a quick, automated diagnosis of Redis latency issues with plain-English explanations and specific recommendations. Enable `latency-monitor-threshold` to start recording events, then run `LATENCY DOCTOR` when you observe performance degradation. Pair it with `LATENCY HISTORY` and `SLOWLOG` for deeper investigation.
