# How to Interpret Ping Output (TTL, RTT, Packet Loss)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ping, ICMP, IPv4, TTL, RTT, Networking

Description: Learn to read and interpret all fields in ping output including TTL, round-trip time, packet loss, and mdev to diagnose network issues accurately.

Understanding what ping output actually means transforms it from a simple "up or down" tool into a powerful diagnostic instrument. Each field tells you something specific about the path between you and the target.

## Anatomy of a Ping Reply

```
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=12.3 ms

   ^           ^              ^          ^          ^
   |           |              |          |          |
Payload     Source IP    Sequence    Time-to-    Round-trip
  size                   number      Live        time (ms)
```

## Interpreting TTL

TTL (Time To Live) decrements at each hop. The remaining TTL hints at the number of hops used:

```bash
ping -c 1 8.8.8.8
# ttl=118 → started at 128 (Windows default), used 10 hops

# Common starting TTL values:
#   64  — Linux, Android, macOS (older)
#   128 — Windows
#   255 — Cisco IOS, some Linux

# Calculate hop count:
#   128 - 118 = 10 hops to 8.8.8.8 from Windows source
#   64 - 52 = 12 hops if source started at 64

# Inconsistent TTL between replies = asymmetric routing or load balancing
ping -c 5 1.1.1.1
# If ttl alternates between 118 and 119 → traffic is being load-balanced
```

## Interpreting Round-Trip Time (RTT)

RTT measures total time from sending the ICMP request to receiving the reply:

```
RTT Range       Assessment
-----------     ----------------------------------------
< 1 ms          Same machine or LAN switch (excellent)
1-10 ms         Local LAN or same datacenter
10-50 ms        Same country/region (typical)
50-150 ms       Cross-continent
150-300 ms      Intercontinental (US to Asia)
> 300 ms        Very long distance, satellite, or problem
```

```bash
# The statistics line shows:
# rtt min/avg/max/mdev = 11.2/12.4/14.1/0.9 ms
#         ^       ^       ^      ^
#       fastest  mean  slowest  jitter

# mdev (mean deviation) = jitter — consistency of latency
# mdev < 1ms: very stable connection
# mdev > 10ms: high jitter (bad for VoIP/video)
```

## Interpreting Packet Loss

```bash
ping -c 100 10.0.0.1
# 100 packets transmitted, 97 received, 3% packet loss

# Interpreting loss percentage:
# 0%         — Perfect
# 0.1-1%     — Minor congestion or CRC errors on link
# 1-5%       — Significant congestion, check link utilization
# 5-20%      — Serious problem — check hardware errors
# > 20%      — Link failing or heavily congested
# 100%       — Host down, routing broken, or firewall blocking

# Intermittent loss pattern (not consecutive):
# seq=1 OK, seq=2 OK, seq=3 LOST, seq=4 OK, seq=5 OK
# → Random loss = congestion or wireless interference

# Consecutive loss pattern:
# seq=1-4 OK, seq=5-10 LOST, seq=11-15 OK
# → Burst loss = cable fault, interface flap, or hardware problem
```

## Interpreting Latency Spikes

```bash
# Variable RTT in same ping session indicates:
# 64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=1.2 ms
# 64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=45.1 ms  ← SPIKE
# 64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=1.3 ms

# Single spike: CPU busy on target, scheduler delay
# Regular spikes: bufferbloat (large queue building and draining)
# Increasing RTT: queue filling over time (congestion)
```

## Detecting Path Changes

```bash
# If TTL changes between replies, the route changed
ping -c 20 8.8.8.8 | grep ttl

# TTL variation indicates:
# - Load balancing across multiple paths
# - BGP route flap
# - Routing protocol reconvergence
```

Reading ping output diagnostically — not just "are packets coming back" — lets you pinpoint whether problems are local (high RTT from the start), mid-path (increasing RTT), or at the destination (loss with low RTT).
