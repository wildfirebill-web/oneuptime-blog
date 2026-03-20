# How to Profile IPv6 Network Stack Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Profiling, Linux, perf, eBPF, Networking, Performance

Description: Profile the Linux IPv6 network stack using perf, eBPF/bpftrace, and kernel tracepoints to identify bottlenecks in packet processing pipelines.

## Introduction

When IPv6 throughput or latency is unexpectedly poor, profiling the kernel network stack reveals where CPU time is spent. Tools like `perf`, `bpftrace`, and `ss` provide deep visibility into packet processing paths.

## Step 1: Measure Overall Network Stack CPU Usage

```bash
# Profile kernel CPU time for 10 seconds during a network load
sudo perf top -a -g \
  --call-graph dwarf \
  -p $(pgrep softirq) \
  -- sleep 10

# Or use perf stat to count events
sudo perf stat -e \
  net:net_dev_xmit,net:netif_receive_skb,\
  net:napi_poll,skb:kfree_skb \
  -a sleep 5
```

## Step 2: Trace IPv6 Packet Processing with perf

```bash
# Record a performance profile during an iperf3 IPv6 test
sudo perf record -a -g -F 99 sleep 30 &
iperf3 -6 -c 2001:db8::1 -t 25 -P 4
wait

# Generate a flame graph
sudo perf script | \
  stackcollapse-perf.pl | \
  flamegraph.pl > ipv6_flamegraph.svg

# Top functions in the IPv6 receive path
sudo perf report --sort comm,dso,sym | \
  grep -E "ip6_|tcp_v6|napi|ixgbe"
```

## Step 3: Trace IPv6 Drops with kprobes

```bash
# Install bpftrace (Ubuntu/Debian)
sudo apt-get install bpftrace

# Trace IPv6 packet drops in real time
sudo bpftrace -e '
kprobe:kfree_skb {
    @drops[kstack] = count();
}
interval:s:5 {
    print(@drops);
    clear(@drops);
}'
```

## Step 4: Monitor Socket-Level Performance

```bash
# Show per-socket IPv6 statistics with extended info
ss -6 -t -i -e

# Key fields to watch:
# rtt: current RTT
# cwnd: congestion window (higher = better throughput)
# retrans: retransmission count (should be near zero)
# rcv_space: receive buffer space used

# Show all IPv6 TCP sockets with queue depths
ss -6 -t -n | awk 'NR>1 {print $2, $3, $5, $6}'
# Columns: state, recv-Q, send-Q, local, peer
```

## Step 5: Use eBPF to Profile IPv6 Receive Latency

```python
#!/usr/bin/env python3
# ipv6_rx_latency.py — measure time from NIC interrupt to application recv

from bcc import BPF

bpf_program = """
#include <uapi/linux/ptrace.h>
#include <net/sock.h>
#include <bcc/proto.h>

BPF_HASH(start_time, u64, u64);
BPF_HISTOGRAM(rx_latency, u64);

// Record timestamp when packet arrives at softirq
TRACEPOINT_PROBE(net, netif_receive_skb) {
    u64 key = (u64)args->skbaddr;
    u64 ts = bpf_ktime_get_ns();
    start_time.update(&key, &ts);
    return 0;
}

// Measure latency when socket receives data
TRACEPOINT_PROBE(net, inet_sock_set_state) {
    u64 key = (u64)args->skaddr;
    u64 *tsp = start_time.lookup(&key);
    if (tsp) {
        u64 latency_ns = bpf_ktime_get_ns() - *tsp;
        // Bucket in microseconds
        rx_latency.increment(bpf_log2l(latency_ns / 1000));
        start_time.delete(&key);
    }
    return 0;
}
"""

b = BPF(text=bpf_program)
print("Tracing IPv6 RX latency... Ctrl-C to stop")

import time
try:
    time.sleep(30)
except KeyboardInterrupt:
    pass

print("\nIPv6 RX latency histogram (microseconds):")
b["rx_latency"].print_log2_hist("usec")
```

## Step 6: Profile Retransmission Causes

```bash
# Check TCP retransmission statistics
netstat -s | grep -i "retransmit\|segment"

# Trace retransmission events with bpftrace
sudo bpftrace -e '
kprobe:tcp_retransmit_skb {
    @[comm] = count();
}
END { print(@); }'

# Monitor dropped packets at the NIC level
ethtool -S eth0 | grep -i "drop\|miss\|error"
```

## Conclusion

IPv6 network stack profiling combines `perf` flame graphs for CPU hotspots, `bpftrace` for kernel-level event tracing, and `ss` for socket-level visibility. Identifying whether bottlenecks are in the NIC driver, softirq processing, or socket receive path guides targeted optimization. Feed profiling results into OneUptime dashboards to correlate stack performance with application-level latency.
