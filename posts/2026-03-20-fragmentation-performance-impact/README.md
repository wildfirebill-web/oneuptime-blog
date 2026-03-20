# How to Understand the Performance Impact of IPv4 Fragmentation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Fragmentation, Performance, MTU, Linux, Networking

Description: Measure and understand the CPU, throughput, and reliability impact of IPv4 fragmentation, and quantify the benefit of avoiding fragmentation through proper MTU sizing.

## Introduction

IPv4 fragmentation has real performance costs that are often underestimated. Every fragment that must be reassembled consumes CPU cycles and memory. When a single fragment is lost, the entire original packet fails - increasing the effective loss rate exponentially. In high-throughput environments, fragmentation can reduce throughput by 20-40% compared to properly sized packets.

## Why Fragmentation Hurts Performance

```text
Performance costs of fragmentation:

1. Increased loss probability:
   A packet split into N fragments has N times the failure chance
   Each fragment can be independently lost
   1% link loss rate + 3-fragment packet = ~3% packet loss
   (vs 1% with unfragmented packets)

2. CPU overhead at receiver:
   Each fragment must be queued, tracked, and reassembled
   Requires memory allocation per fragment set
   Reassembly lock contention on multi-core systems

3. Memory pressure:
   Fragments held in memory until all arrive (or ipfrag_time expires)
   Stale incomplete fragment sets accumulate on lossy networks

4. Doubled throughput cost:
   IP+UDP = 28 bytes overhead on first fragment
   IP only = 20 bytes overhead on subsequent fragments
   vs single large packet with just 28 bytes overhead
```

## Measure Fragmentation Impact

```bash
# Benchmark 1: Compare throughput with and without fragmentation

# Setup: use tc netem to simulate a link with specific MTU

tc qdisc add dev eth0 root netem mtu 1400 delay 1ms
# This forces all packets > 1400 to be fragmented at this host

# Measure throughput WITH fragmentation:
iperf3 -c 10.20.0.5 -t 30 -u -b 500M -l 8960  # Large UDP packets
# All 8960-byte UDP packets will be fragmented into 7 fragments

# Measure throughput WITHOUT fragmentation:
tc qdisc del dev eth0 root  # Remove fragmentation limit
iperf3 -c 10.20.0.5 -t 30 -u -b 500M -l 1400  # Packets that fit in MTU

# Compare the receiver-side throughput numbers
```

## Measure CPU Impact of Fragmentation

```bash
# Monitor CPU during fragment-heavy traffic:

# Start fragment-generating traffic:
ping -s 8000 -f 10.20.0.5 &   # Flood ping with large (fragmented) packets
PING_PID=$!

# Measure CPU:
mpstat 1 5 | tail -5

# Kill ping:
kill $PING_PID

# Comparison: same traffic with unfragmented packets:
ping -s 1400 -f 10.20.0.5 &
PING_PID=$!
mpstat 1 5 | tail -5
kill $PING_PID

# The fragmented traffic should show higher %sys CPU usage
```

## Monitor Fragmentation Statistics

```bash
# Track fragmentation impact in production:

# Check fragmentation rate:
watch -n 1 "nstat | grep -E 'IpFrag|IpReasm'"
# IpFragCreates: fragments we create (we're fragmenting)
# IpReasmReqds: fragments waiting for reassembly (we're receiving fragments)
# IpReasmOKs: successfully reassembled
# IpReasmFails: FAILED reassembly (lost fragments!)

# Calculate failure rate:
nstat | python3 -c "
import sys
data = {}
for line in sys.stdin:
    parts = line.split()
    if len(parts) >= 2:
        data[parts[0]] = int(parts[1])

ok = data.get('IpReasmOKs', 0)
fail = data.get('IpReasmFails', 0)
total = ok + fail
if total > 0:
    fail_rate = fail / total * 100
    print(f'Reassembly failure rate: {fail_rate:.2f}% ({fail}/{total})')
"
```

## Practical Loss Rate Calculation

```python
#!/usr/bin/env python3
# Calculate effective packet loss with fragmentation

def calculate_loss_with_fragmentation(link_loss_pct, packet_size_bytes, mtu=1500):
    """
    Calculate effective packet loss when packets are fragmented.

    With fragmentation, each original packet is split into N fragments.
    The original packet fails if ANY fragment is lost.
    """
    link_loss = link_loss_pct / 100

    # Number of fragments per packet:
    if packet_size_bytes <= mtu:
        n_fragments = 1
    else:
        import math
        # Account for IP headers on each fragment
        data_per_fragment = mtu - 20  # IP header
        n_fragments = math.ceil(packet_size_bytes / data_per_fragment)

    # Probability all N fragments arrive successfully:
    success_prob = (1 - link_loss) ** n_fragments
    effective_loss = (1 - success_prob) * 100

    return n_fragments, effective_loss

# Example: 0.5% link loss, various packet sizes
print("Packet Size | Fragments | Effective Loss (0.5% link)")
print("-----------|-----------|---------------------------")
for size in [1000, 1500, 3000, 9000, 16000]:
    frags, loss = calculate_loss_with_fragmentation(0.5, size)
    print(f"{size:10d} | {frags:9d} | {loss:.3f}%")
```

## Conclusion

Fragmentation multiplies your effective packet loss rate - a 1% link loss becomes ~7% effective loss for a 7-fragment packet. It also adds CPU overhead for reassembly and memory pressure from queued fragment sets. The performance fix is simple: size packets to fit within the path MTU. For TCP, this happens automatically via PMTUD and MSS negotiation. For UDP, limit application payload to the path MTU minus 28 bytes. Monitor `IpReasmFails` in production - any non-zero rate indicates fragmentation causing dropped packets.
