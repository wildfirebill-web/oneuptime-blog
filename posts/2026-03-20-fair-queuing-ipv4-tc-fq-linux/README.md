# How to Set Up Fair Queuing for IPv4 Traffic with tc fq on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tc, Fair Queuing, IPv4, Linux, QoS, Fq_codel

Description: Configure Linux tc fair queuing qdiscs (fq, fq_codel, CAKE) to ensure equitable bandwidth distribution among IPv4 flows without per-IP configuration.

Fair queuing distributes bandwidth equally among competing flows without requiring manual per-IP or per-port rules. This prevents any single flow (like a large download) from starving other connections.

## Available Fair Queuing qdiscs

| qdisc | Description | Best Use |
|---|---|---|
| `fq` | Per-flow fair queuing with pacing | High-performance servers |
| `fq_codel` | FQ + CoDel AQM (reduces bufferbloat) | General purpose |
| `cake` | Modern all-in-one (FQ + shaping + AQM) | Edge routers, home gateways |

## Using fq (Fair Queue)

```bash
# Replace the default pfifo_fast qdisc with fq

sudo tc qdisc replace dev eth0 root fq

# Configure with custom parameters
sudo tc qdisc replace dev eth0 root fq \
  flow_limit 100 \       # Max packets queued per flow
  quantum 3028 \         # Bytes sent per round-robin turn
  initial_quantum 15140  # Initial quantum for new flows (TCP slow start boost)

# View statistics
sudo tc -s qdisc show dev eth0
```

## Using fq_codel (Most Commonly Recommended)

```bash
# Apply fq_codel with defaults (good for most use cases)
sudo tc qdisc replace dev eth0 root fq_codel

# With custom parameters
sudo tc qdisc replace dev eth0 root fq_codel \
  limit 10240 \      # Queue size in packets
  flows 1024 \       # Number of flow buckets (power of 2)
  target 5ms \       # Target queuing delay (CoDel target)
  interval 100ms     # CoDel interval

# Check for drops and ECN marks
sudo tc -s qdisc show dev eth0
```

## Using CAKE (Recommended for Edge Routers)

CAKE (Common Applications Kept Enhanced) combines fair queuing with rate limiting and AQM:

```bash
# Install CAKE (may need iproute2 upgrade)
sudo apt install iproute2 -y

# Apply CAKE with a bandwidth limit (e.g., for a 100 Mbps link)
sudo tc qdisc replace dev eth0 root cake bandwidth 95mbit

# CAKE with additional options
sudo tc qdisc replace dev eth0 root cake \
  bandwidth 95mbit \     # Set slightly below your ISP's rate
  diffserv3 \           # Use 3-tier DSCP prioritization
  dual-dsthost          # Fair sharing per destination host
```

## Combining fq_codel with HTB Classes

For bandwidth guarantees plus fair queuing within each class:

```bash
# HTB provides class-based bandwidth guarantees
sudo tc qdisc add dev eth0 root handle 1: htb default 10
sudo tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit
sudo tc class add dev eth0 parent 1:1 classid 1:10 htb rate 100mbit

# fq_codel on the leaf class provides fair queuing within the class
sudo tc qdisc add dev eth0 parent 1:10 handle 10: fq_codel
```

## Verifying Flow Fairness

```bash
# Generate multiple flows (4 parallel streams)
iperf3 -c <SERVER_IP> -P 4 -t 30

# Monitor per-flow bandwidth in real time
sudo iftop -i eth0 -n

# View qdisc stats including drops and backlog
sudo tc -s qdisc show dev eth0
# Key fields: backlog (current queue size), drops, ecn_mark
```

## Checking for Bufferbloat

```bash
# Measure bufferbloat impact (should be near baseline with fq_codel/CAKE)
# Run a download and simultaneously ping
wget -O /dev/null http://speedtest.net/largefile &
ping -c 50 8.8.8.8
# With fq_codel, ping RTT should remain low even during download
```

Fair queuing qdiscs are drop-in improvements over the default `pfifo_fast` - simply replacing the default qdisc with `fq_codel` or CAKE substantially reduces latency during congestion.
