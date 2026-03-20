# How to Troubleshoot IPv6 Slow Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Performance, Troubleshooting, MTU, TCP, Network Diagnostics

Description: Diagnose why IPv6 is slower than IPv4 on the same network, including MTU black holes, suboptimal routing paths, TCP congestion window issues, and Happy Eyeballs delays.

## Introduction

IPv6 traffic should perform similarly to IPv4 on a properly configured network. When IPv6 is noticeably slower, common causes include MTU black holes (packets silently dropped), longer routing paths, TCP slow start issues, or PMTUD failures. This guide provides tools to measure and diagnose IPv6 performance problems.

## Step 1: Measure IPv4 vs IPv6 Performance

```bash
# Compare download speed over IPv4 vs IPv6
echo "IPv4 speed:"
time curl -4 -s -o /dev/null https://ipv4.speedtest.net/test/

echo ""
echo "IPv6 speed:"
time curl -6 -s -o /dev/null https://ipv6.speedtest.net/test/

# More accurate comparison with iperf3
# Server:
iperf3 -s

# Client - IPv4:
iperf3 -c server.example.com -4 -t 10

# Client - IPv6:
iperf3 -c server.example.com -6 -t 10
```

## Step 2: Measure Latency Difference

```bash
# Compare ping latency
echo "IPv4 latency to Google:"
ping -c 10 8.8.8.8 | tail -3

echo ""
echo "IPv6 latency to Google:"
ping6 -c 10 2001:4860:4860::8888 | tail -3

# Use mtr for path-level comparison
echo "IPv4 path:"
mtr --report --ipv4 google.com

echo ""
echo "IPv6 path:"
mtr --report --ipv6 google.com

# Compare hop counts and latency at each hop
```

## Step 3: Test for MTU Issues (Most Common Cause)

```bash
# Test with progressively larger packets
echo "Packet size test over IPv6:"
for size in 576 1000 1280 1400 1452 1500; do
    result=$(ping6 -c 3 -s $size 2001:4860:4860::8888 2>&1)
    if echo "$result" | grep -q "3 received"; then
        echo "  $size bytes: OK"
    else
        avg=$(echo "$result" | grep "avg" | awk -F'/' '{print $5}')
        echo "  $size bytes: FAIL/PARTIAL (avg: ${avg}ms)"
    fi
done

# If large packets fail or are slow: MTU black hole
# Fix: allow ICMPv6 Packet Too Big through firewalls
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type 2 -j ACCEPT
```

## Step 4: Check TCP Performance

```bash
# Check TCP congestion window with ss
ss -tin6 'dst 2001:4860:4860::8888' | grep cwnd

# View detailed TCP metrics for an IPv6 connection
ss -tin6 state established | head -20

# Key metrics:
# cwnd: congestion window (larger = better throughput)
# rtt: round-trip time
# retrans: retransmissions (if high, indicates packet loss)
# rcvbuf/sndbuf: socket buffers

# Check TCP buffer sizes
sysctl net.core.rmem_max net.core.wmem_max
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem  # Applies to IPv6 too
```

## Step 5: Analyze Routing Path

```bash
# Is IPv6 taking a longer path than IPv4?
echo "IPv4 path to Google:"
traceroute -n 8.8.8.8 | wc -l

echo "IPv6 path to Google:"
traceroute6 -n 2001:4860:4860::8888 | wc -l

# Compare paths in detail
traceroute -n 8.8.8.8
traceroute6 -n 2001:4860:4860::8888

# IPv6 traffic may be tunneled over IPv4, adding latency
# Look for ::ffff:IPv4 hops in traceroute6 output
```

## Step 6: Check TCP Segmentation Offload

```bash
# On some systems, IPv6 TCP segmentation offload (TSO) may be suboptimal
ethtool -k eth0 | grep "tcp-segmentation-offload\|generic-segmentation\|large-receive"

# Try disabling TSO for IPv6 to test if it improves performance
# (only for testing — measure before and after)
sudo ethtool -K eth0 tso off
sudo ethtool -K eth0 gso off
iperf3 -c server -6 -t 10
sudo ethtool -K eth0 tso on
sudo ethtool -K eth0 gso on
```

## Step 7: Happy Eyeballs Delay Impact

```bash
# If applications are slow to start over IPv6:
# Check if Happy Eyeballs fallback delay is too long
# Default is 250ms in RFC 8305

# Test with curl and measure connection time
curl -6 -w "Connect: %{time_connect}s\nTTFB: %{time_starttransfer}s\n" \
    -s -o /dev/null https://example.com

curl -4 -w "Connect: %{time_connect}s\nTTFB: %{time_starttransfer}s\n" \
    -s -o /dev/null https://example.com

# If IPv6 connect time is much higher, check:
# 1. DNS response time for AAAA vs A
# 2. SYN timeout before fallback to IPv4
```

## Conclusion

IPv6 slow performance is most commonly caused by MTU black holes (test with large ping6 packets), longer routing paths (compare traceroute hop counts), or TCP retransmissions. Check with `iperf3` to measure raw throughput, `mtr` to compare paths, and large `ping6 -s 1400` tests to identify MTU issues. Allow ICMPv6 type 2 (Packet Too Big) through all firewalls to enable proper PMTUD. In cloud environments, IPv6 traffic may traverse additional encapsulation layers that don't affect IPv4, adding latency.
