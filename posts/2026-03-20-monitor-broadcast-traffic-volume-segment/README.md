# How to Monitor Broadcast Traffic Volume on a Network Segment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Broadcast, Network Monitoring, Linux, tcpdump, iftop

Description: Measure and monitor the volume of broadcast traffic on a network segment using tcpdump, iftop, nload, and interface statistics to detect storms and excessive broadcast rates.

## Introduction

Excessive broadcast traffic degrades performance for every host on a segment, consuming CPU and bandwidth even for hosts that do not care about the broadcast content. Monitoring broadcast volume helps establish a baseline and detect storms early.

## Using Interface Counters

The fastest check is reading broadcast counters from the interface statistics:

```bash
# Show broadcast RX/TX counters for eth0
ip -s link show eth0

# Watch counters update every second
watch -n 1 "ip -s link show eth0 | grep -A2 'RX\|TX'"
```

The output includes `bcast:` counters showing total broadcast packets received and transmitted.

## Calculating Broadcast Rate with awk

```bash
#!/bin/bash
# Calculate broadcast packets per second over 10 seconds

IFACE="eth0"
STAT_FILE="/sys/class/net/$IFACE/statistics"

START=$(cat "$STAT_FILE/rx_frame_errors" 2>/dev/null || echo 0)
# Use multicast as proxy for broadcast+multicast combined
START_BC=$(ip -s link show "$IFACE" | awk '/RX:/{getline; print $4}')

sleep 10

END_BC=$(ip -s link show "$IFACE" | awk '/RX:/{getline; print $4}')
DIFF=$(( END_BC - START_BC ))
echo "Broadcast/multicast packets in 10s: $DIFF ($(( DIFF / 10 )) pps)"
```

## Counting Broadcasts with tcpdump

```bash
# Count all broadcast packets in 30 seconds
sudo timeout 30 tcpdump -i eth0 -n -q "broadcast or dst 255.255.255.255" 2>&1 | tail -3
```

Output example:
```
1423 packets captured
1423 packets received by filter
0 packets dropped by kernel
```

## Breaking Down Broadcast by Protocol

```bash
# Count ARP broadcasts
sudo timeout 10 tcpdump -i eth0 -n -q "arp and broadcast" 2>&1 | tail -1

# Count DHCP broadcasts (UDP port 67/68)
sudo timeout 10 tcpdump -i eth0 -n -q "udp and (dst port 67 or dst port 68)" 2>&1 | tail -1

# Count NetBIOS broadcasts
sudo timeout 10 tcpdump -i eth0 -n -q "udp and dst port 137 and broadcast" 2>&1 | tail -1
```

## Top Broadcast Senders

```bash
# Find which hosts are generating the most broadcasts
sudo tcpdump -i eth0 -n "broadcast" -c 500 2>/dev/null | \
  awk '{print $3}' | sort | uniq -c | sort -rn | head -10
```

## Graphing Broadcast Traffic with vnstat

```bash
# Install vnstat for per-interface traffic graphing
sudo apt install vnstat
sudo vnstat -i eth0 --live

# View hourly/daily stats
vnstat -i eth0 -h
```

## Setting a Baseline and Alerting

Normal broadcast traffic depends on the network, but a rough guide:
- **< 50 pps**: healthy
- **50–500 pps**: moderate (many DHCP renewals, ARP on busy segment)
- **> 500 pps**: investigate — possible storm or misbehaving host
- **> 10,000 pps**: likely a broadcast storm — take immediate action

## Conclusion

Monitor broadcast volume using interface counters for continuous tracking and `tcpdump` for protocol-level breakdown. Establish a baseline for your network, then alert on deviations to catch storms before they take down the entire segment.
