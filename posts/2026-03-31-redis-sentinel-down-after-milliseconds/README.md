# How to Configure down-after-milliseconds in Redis Sentinel

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sentinel, Configuration, Failover, Timeout

Description: Learn how to tune down-after-milliseconds in Redis Sentinel to balance failover speed against false-positive detection in your environment.

---

`down-after-milliseconds` is the time a Sentinel waits without a PING response before marking a Redis node as subjectively down (S-DOWN). Getting this value right balances failover speed against false positives.

## What It Controls

Sentinels send periodic PINGs to each monitored Redis instance. If a node does not respond within `down-after-milliseconds`, the Sentinel marks it S-DOWN. When enough Sentinels agree (quorum), the node becomes O-DOWN and failover begins.

```bash
# sentinel.conf
sentinel down-after-milliseconds mymaster 5000
# 5 seconds without response -> S-DOWN
```

## Default and Recommended Values

```text
Default: 30000ms (30 seconds)

Conservative (less false positives):
  10000-30000ms  - cloud environments with variable latency

Moderate (balanced):
  5000ms         - typical LAN or fast datacenter

Aggressive (fast failover):
  1000-2000ms    - high-confidence networks (colocated servers)
```

## Risks of Setting Too Low

On a busy server, a 1-second GC pause or kernel scheduling delay could trigger a false failover:

```text
Primary busy processing a large SCAN
Sentinel PING goes unanswered for 1500ms
-> False S-DOWN -> unnecessary failover
-> Client reconnection overhead for no reason
```

## Risks of Setting Too High

With 30 seconds, your application is unavailable for 30+ seconds waiting for failover:

```text
Primary crashes at T=0
Sentinel marks S-DOWN at T=30
Quorum reached at T=30
Failover completes at T=32
Total downtime: ~32 seconds
```

## Applying the Setting

```bash
# sentinel.conf
sentinel monitor mymaster 192.168.1.10 6379 2
sentinel down-after-milliseconds mymaster 5000
```

Change at runtime:

```bash
redis-cli -p 26379 SENTINEL SET mymaster down-after-milliseconds 3000
```

Verify:

```bash
redis-cli -p 26379 SENTINEL masters
# Look for "down-after-milliseconds" in output
```

## This Setting Affects Replicas Too

`down-after-milliseconds` applies to replicas as well. If a replica is slow, Sentinel will mark it as S-DOWN too. This impacts which replicas are eligible for promotion:

```bash
# Check replica status
redis-cli -p 26379 SENTINEL replicas mymaster
# s_down flag indicates the replica is considered down
```

## Tuning Strategy

Test your network latency first:

```bash
# Measure actual PING response time to your Redis nodes
redis-cli -p 6379 PING
# Check how long it takes under load

# Also monitor:
redis-cli INFO stats | grep instantaneous_ops_per_sec
```

Then set `down-after-milliseconds` to at least 3-5x your measured P99 PING latency under peak load.

## Summary

`down-after-milliseconds` in Redis Sentinel controls how quickly a non-responsive node is marked as down. Set it low (1-5 seconds) for fast failover in reliable networks, or higher (10-30 seconds) in environments with variable latency. Always test your chosen value under realistic load before deploying to production.
