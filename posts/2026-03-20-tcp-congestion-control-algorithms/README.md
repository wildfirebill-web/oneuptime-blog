# How to Understand TCP Congestion Control Algorithms (Reno, CUBIC, BBR)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Congestion Control, BBR, CUBIC, Linux, Performance

Description: Compare TCP congestion control algorithms including Reno, CUBIC, and BBR, and understand how each one manages network congestion and maximizes throughput.

## Introduction

TCP congestion control algorithms determine how fast a sender transmits data and how it responds to network congestion. The algorithm significantly impacts throughput, latency, and fairness — especially on long-distance or lossy links. Understanding the tradeoffs between Reno, CUBIC, and BBR helps you choose the right algorithm for your workload.

## How Congestion Control Works

All TCP congestion control algorithms manage the Congestion Window (CWND) — the maximum amount of unacknowledged data the sender can have in flight:

```
Throughput ≈ min(CWND, Receive Window) / RTT

Algorithms differ in:
1. How fast they grow CWND (slow start vs congestion avoidance)
2. How they detect congestion (packet loss vs delay signals)
3. How aggressively they reduce CWND after congestion
```

## TCP Reno (Classic)

```
Algorithm behavior:
1. Slow Start: CWND doubles each RTT (exponential growth)
2. At threshold: switches to Congestion Avoidance (+1 MSS per RTT)
3. On loss (3 duplicate ACKs): CWND = CWND / 2 (Fast Recovery)
4. On timeout: CWND = 1 MSS (Slow Start restart)

Weakness: Very conservative on high-bandwidth × high-latency paths
          CWND/2 reduction wastes most of the available pipe
```

## TCP CUBIC (Linux Default)

```
Algorithm behavior:
1. Uses a cubic function to grow CWND (independent of RTT)
2. On congestion: CWND × 0.8 (20% reduction, less aggressive than Reno)
3. Probes back to previous window quickly using cubic curve
4. Scales well with very high bandwidth links

# Good for: high-bandwidth LAN/datacenter, standard internet connections
# Weakness: still loss-based, can be slow to recover on lossy links
```

## TCP BBR (Bottleneck Bandwidth and RTT)

```
Algorithm behavior:
1. Estimates actual bottleneck bandwidth by measuring delivery rate
2. Estimates minimum RTT to find queue-free path
3. Sets CWND and pacing rate based on bandwidth × RTT model
4. Does NOT react to packet loss as primary signal (avoids over-slowing)

# Good for: high-latency WAN, satellite links, congested networks
# Can be unfair to CUBIC/Reno connections at the same bottleneck
```

## Comparing in Practice

```bash
# Check available algorithms
sysctl net.ipv4.tcp_available_congestion_control
# cubic bbr reno ...

# View current default
sysctl net.ipv4.tcp_congestion_control

# Test each algorithm with iperf3
sysctl -w net.ipv4.tcp_congestion_control=cubic
iperf3 -c 10.20.0.5 -t 30
echo "CUBIC done"

sysctl -w net.ipv4.tcp_congestion_control=bbr
iperf3 -c 10.20.0.5 -t 30
echo "BBR done"

sysctl -w net.ipv4.tcp_congestion_control=reno
iperf3 -c 10.20.0.5 -t 30
echo "Reno done"
```

## When to Use Each Algorithm

| Scenario | Recommended |
|---|---|
| Standard datacenter (low RTT, low loss) | CUBIC |
| High-bandwidth WAN (>50ms RTT) | BBR |
| Satellite / high-latency (>200ms RTT) | BBR |
| Lossy wireless networks | BBR |
| Bulk file transfer between servers | BBR |
| Real-time gaming / VoIP | BBR or CUBIC |
| Legacy compatibility | CUBIC |

## Conclusion

CUBIC is the right default for most LAN and datacenter scenarios. BBR excels on high-latency or lossy paths where loss-based algorithms react too conservatively. The practical performance difference between CUBIC and BBR can be dramatic on paths with 100ms+ RTT — BBR can achieve 5-10× higher throughput by not over-reacting to loss signals.
