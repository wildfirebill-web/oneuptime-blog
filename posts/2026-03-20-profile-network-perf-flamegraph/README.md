# How to Profile Network Performance with perf and Flamegraphs on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: perf, Flamegraph, Linux, Network Performance, Profiling, Kernel

Description: Learn how to use Linux perf and Flamegraph to identify CPU bottlenecks in the network stack, NIC drivers, and application code during high-throughput network workloads.

## Why Profile Network Performance?

When network throughput plateaus below the theoretical maximum, the bottleneck could be:
- The kernel TCP stack consuming excessive CPU
- NIC driver interrupt handling
- Application code that's slow to read/write sockets
- Memory copies in the network path

Linux `perf` captures CPU stack traces at high frequency, and Flamegraphs visualize where time is spent — pinpointing the exact function causing the bottleneck.

## Step 1: Install Prerequisites

```bash
# Install perf (Linux performance profiler)
sudo apt-get install -y linux-tools-$(uname -r) linux-tools-generic

# Verify perf is working
perf stat -e task-clock echo "hello"

# Install Flamegraph scripts
git clone https://github.com/brendangregg/FlameGraph /opt/flamegraph
export PATH=/opt/flamegraph:$PATH

# Install dependency for symbol resolution
sudo apt-get install -y binutils
```

## Step 2: Start a Network Workload

Run the network workload you want to profile while capturing:

```bash
# Terminal 1: Start iperf3 server
iperf3 -s -p 5201 &

# Terminal 2: Start iperf3 client (this will be the workload)
# We'll capture while this runs
iperf3 -c 127.0.0.1 -t 60 -P 4 &

# Note the iperf3 client PID for targeted profiling
CLIENT_PID=$(pgrep iperf3 | tail -1)
echo "iperf3 client PID: $CLIENT_PID"
```

## Step 3: Capture perf Data

```bash
# System-wide capture for 30 seconds at 99 Hz
# (use 99 Hz instead of 100 Hz to avoid lockstep with timer ticks)
sudo perf record -F 99 -a -g -- sleep 30

# Or capture only the target process
sudo perf record -F 99 -p $CLIENT_PID -g -- sleep 30

# Kernel-only profiling (to find kernel TCP stack bottlenecks)
sudo perf record -F 99 -a -g -e cpu-clock -- sleep 30

# Verify capture
ls -lh perf.data
```

## Step 4: Generate a Flamegraph

```bash
# Convert perf data to folded stack format
sudo perf script | /opt/flamegraph/stackcollapse-perf.pl > /tmp/stacks.folded

# Generate the flamegraph SVG
/opt/flamegraph/flamegraph.pl /tmp/stacks.folded > /tmp/network-flamegraph.svg

# Open in browser
# On desktop: xdg-open /tmp/network-flamegraph.svg
# On server: copy to local machine
scp user@server:/tmp/network-flamegraph.svg .
```

## Step 5: Interpret Network Flamegraphs

Key patterns to look for in a network flamegraph:

```
Common bottlenecks:

1. Wide "tcp_sendmsg" or "tcp_recvmsg" bars
   → CPU spent in TCP send/receive path
   → May indicate missing TSO/GRO offload

2. Wide "ixgbe_poll" or "mlx5_rx" (driver names)
   → NIC driver consuming CPU
   → May indicate ring buffer drops or missing RSS

3. Wide "memcpy" or "copy_user"
   → Memory copies in network path
   → Consider using zero-copy (sendfile, splice)

4. Wide "skb_copy" or "skb_clone"
   → Socket buffer copies
   → May indicate packet cloning overhead

5. Wide "__napi_poll"
   → NAPI polling overhead
   → Consider increasing NAPI budget
```

## Step 6: Profile Specific Network Events

```bash
# Profile network-specific events
# Measure receive packet processing
sudo perf record -e net:netif_receive_skb -a sleep 10
sudo perf report --stdio | head -30

# Count system calls in iperf3
sudo perf stat -e 'syscalls:sys_enter_*' -p $CLIENT_PID sleep 10

# Count context switches
sudo perf stat -e context-switches,cache-misses -p $CLIENT_PID sleep 10
```

## Step 7: Profile with eBPF (Advanced)

For more detailed analysis without perf overhead:

```bash
# Install bcc tools
sudo apt-get install -y bpfcc-tools

# Trace TCP send/receive latency
sudo tcplife -p $CLIENT_PID

# Trace socket buffer allocation
sudo stackcount -p $CLIENT_PID 'tcp_sendmsg'

# CPU stack trace at socket read time
sudo profile -p $CLIENT_PID 10
```

## Conclusion

Using `perf record -F 99 -a -g` during a network workload and visualizing with FlameGraph reveals exactly which kernel or application functions consume CPU during network operations. Wide bars for TCP functions suggest missing offloads; wide NIC driver bars suggest insufficient RSS queues or ring buffers. This profiling approach is essential for squeezing the last percentages of performance from high-throughput network paths.
