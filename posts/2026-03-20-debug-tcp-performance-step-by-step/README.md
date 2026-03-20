# How to Debug TCP Performance Issues Step by Step

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Performance, Debugging, Linux, Networking, Methodology

Description: Follow a systematic step-by-step methodology to diagnose TCP performance problems by ruling out each layer from physical to application.

## Introduction

TCP performance problems are multi-layered. The same symptom — "transfers are slow" — can have a dozen different causes. A systematic approach rules out each possible cause methodically, avoiding the trap of applying fixes randomly and not knowing what actually helped.

## The Diagnostic Ladder

```
Layer 7 (Application) → is the app the bottleneck?
Layer 4 (TCP)         → window, retransmissions, congestion?
Layer 3 (IP/routing)  → routing, MTU, fragmentation?
Layer 2/1 (Physical)  → duplex, errors, cable quality?
```

Start at Layer 1 and work up.

## Step 1: Physical Layer

```bash
# Check for interface errors (CRC, frame errors = hardware/cable issue)
ip -s link show eth0
# RX errors, dropped, overruns should all be 0
# TX errors should be 0

# Check interface speed and duplex (duplex mismatch = severe performance issue)
ethtool eth0 | grep -E "Speed|Duplex"
# Should show: Speed: 1000Mb/s, Duplex: Full

# Check for link flaps in logs
journalctl -k | grep "eth0.*link"
dmesg | grep "eth0" | grep -i "up\|down"
```

## Step 2: Layer 3 — Routing and MTU

```bash
# Verify correct path and MTU
traceroute -n 10.20.0.5
tracepath 10.20.0.5     # Also shows MTU at each hop

# Check for MTU issues
ping -s 1472 -M do -c 3 10.20.0.5
# If fails: MTU problem (see fragmentation/PMTUD troubleshooting)

# Check for packet loss on the path
ping -c 100 -i 0.2 10.20.0.5 | tail -3
# Any loss% > 0: investigate path
```

## Step 3: TCP Layer — Window and Retransmissions

```bash
# Check current TCP performance metrics
ss -tin state established | grep -E "cwnd|rtt|retrans|snd_wnd"

# Measure actual throughput
iperf3 -c 10.20.0.5 -t 30

# Check retransmission rate
nstat | grep TcpRetrans
iperf3 -c 10.20.0.5 -t 30 -i 5 | grep Retr

# Compare to theoretical maximum
python3 -c "
cwnd_mss = 100  # from ss output
mss = 1460
rtt_ms = 5      # from ss output
theoretical = (cwnd_mss * mss / (rtt_ms/1000)) * 8 / 1e6
print(f'CWND-limited max: {theoretical:.1f} Mbps')
"
```

## Step 4: Application Layer

```bash
# Check if application is the bottleneck
# Profile: is the app spending time in network syscalls or processing?

strace -p $(pgrep app) -e trace=send,recv,write,read -T 2>&1 | \
  awk '/<[0-9]+\.[0-9]+>/{match($0, /<([0-9.]+)>/, t); if(t[1]+0>0.01) print "SLOW SYSCALL:", $0}' | head -20

# Check application-level metrics
# If app has metrics: look at request duration, queue depth, CPU usage
```

## Step 5: Comprehensive ss Diagnostic

```bash
# One command that captures everything relevant
ss -tin state established | head -20

# Parse key values
ss -tin state established | awk '
  /cwnd/{
    match($0, /rtt:([0-9.]+)/, rtt)
    match($0, /cwnd:([0-9]+)/, cwnd)
    match($0, /mss:([0-9]+)/, mss)
    match($0, /retrans:([0-9]+)/, retr)
    match($0, /snd_wnd:([0-9]+)/, win)

    theoretical = (cwnd[1]+0) * (mss[1]+0) / (rtt[1]/1000) * 8 / 1e6
    printf "RTT: %s ms | CWND: %s MSS | Window: %s | Retrans: %s | Max: %.1f Mbps\n",
      rtt[1], cwnd[1], win[1], retr[1], theoretical
  }' | head -5
```

## Conclusion

Systematic TCP debugging starts at the physical layer and works up. Hardware errors trump everything — fix duplex mismatches and interface errors first. MTU issues cause mysterious hangs — test with large DF-set pings. TCP window and retransmission analysis reveals whether the bottleneck is buffer size, packet loss, or congestion. Application profiling catches the cases where the network is fine but the app can't keep up with incoming data.
