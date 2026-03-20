# How to Use ICMP to Diagnose MTU Black Holes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMP, MTU, PMTUD, Networking, IPv4, Troubleshooting

Description: Use ICMP-based techniques to identify and diagnose MTU black holes where packets are silently dropped due to PMTUD failures on the network path.

## Introduction

An MTU black hole is a network condition where a router silently drops packets that exceed the next-hop MTU but cannot send the ICMP Fragmentation Needed message back (because it's blocked by a firewall). The result is TCP connections that establish normally but hang when transferring more than a small amount of data.

## Symptoms of an MTU Black Hole

- TCP three-way handshake succeeds (SYN, SYN-ACK, ACK work fine — small packets)
- HTTP HEAD requests work but GET requests for large files hang
- SSH connects but then hangs when you type a command with long output
- VPN tunnel works for small pings but fails for real traffic
- Web pages partially load (text loads, images time out)

## Step 1: Confirm the MTU Black Hole

```bash
# Test with progressively smaller DF-set packets
# If large packets fail but small ones succeed: MTU black hole

# Start at full MTU minus headers (1472 = 1500 - 20 IP - 8 ICMP)
ping -s 1472 -M do -c 3 10.20.0.5   # Try 1500 byte total
ping -s 1200 -M do -c 3 10.20.0.5   # Try 1228 byte total
ping -s 800  -M do -c 3 10.20.0.5   # Try 828 byte total

# If 1472 fails but 800 succeeds: black hole with path MTU < 1500
# Use tracepath to automatically find the bottleneck MTU:
tracepath 10.20.0.5
```

## Step 2: Find the Black Hole Router

```bash
# Use traceroute with decreasing TTL and large DF-set packets
# to find which hop swallows the large packets

# Send large UDP probe (simulates the problem)
traceroute -n --mtu 10.20.0.5

# Or use pmtud tool
apt install pmtud 2>/dev/null || apt install mtr

# MTR with large packet size to see where it fails
mtr --psize 1400 --report -n 10.20.0.5
# The hop where large packets are lost is the black hole location
```

## Step 3: Verify ICMP Frag Needed is Blocked

```bash
# From the sender, check if Frag Needed messages are arriving
tcpdump -i eth0 -n -v 'icmp[0]=3 and icmp[1]=4' &

# Now send a large DF packet
ping -s 1472 -M do -c 1 10.20.0.5

# If tcpdump shows "need-to-frag" messages: PMTUD is working, black hole is local
# If tcpdump shows NOTHING: the Frag Needed message is being blocked upstream
```

## Step 4: Workarounds

### Fix 1: Clamp TCP MSS (Most Universal Fix)

```bash
# On the router/firewall between networks, clamp MSS to safe value
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

# Or set explicit MSS (safer for VPN environments)
# 1360 is safe for most VPN/tunnel overhead scenarios
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1360
```

### Fix 2: Reduce MTU on the Affected Interface

```bash
# Lower the MTU so packets never trigger the bottleneck
ip link set eth0 mtu 1400

# Or reduce MTU on the VPN/tunnel interface
ip link set tun0 mtu 1350
```

### Fix 3: Allow Frag Needed Through Firewalls

```bash
# Ensure all firewalls on the path pass Type 3 Code 4
iptables -I FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I INPUT   -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I OUTPUT  -p icmp --icmp-type fragmentation-needed -j ACCEPT
```

## Conclusion

MTU black holes are among the most frustrating network problems because they produce inconsistent, hard-to-reproduce failures. The diagnostic path is clear: test with DF-set pings of varying sizes, use tracepath to find the MTU bottleneck, and check for blocked ICMP Frag Needed messages. TCP MSS clamping is the most robust fix for networks where you can't control all the routers in the path.
