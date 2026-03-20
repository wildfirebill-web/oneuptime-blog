# How to Troubleshoot Path MTU Discovery (PMTUD) Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MTU, PMTUD, IPv4, Troubleshooting, Linux, Networking, Firewall

Description: Diagnose and fix Path MTU Discovery failures caused by ICMP blocking, resulting in TCP connections that establish but hang when sending large data.

## Introduction

PMTUD (Path MTU Discovery) is the mechanism by which TCP dynamically discovers the maximum packet size that can traverse a network path without fragmentation. When it fails - because ICMP messages are blocked - TCP connections appear to work for small data but hang or drop when sending larger payloads. This is called an "MTU black hole" and is one of the most perplexing network problems.

## Symptoms of PMTUD Failure

```text
Classic PMTUD failure symptoms:
- TCP connects successfully (SYN/SYN-ACK small packets work)
- Small responses work (HTTP headers, initial data)
- Large responses hang or fail (HTTP body, file downloads)
- SSH connects but becomes unresponsive during data transfer
- Web pages load for text but images don't load
- Applications work on most networks but fail behind specific firewalls
```

## Diagnose PMTUD Black Hole

```bash
# Step 1: Test if large packets reach the destination

ping -M do -s 1400 10.20.0.5 && echo "1400 works"
ping -M do -s 1472 10.20.0.5 && echo "1472 works"
# If smaller sizes work but larger fail: MTU bottleneck found

# Step 2: Find the path MTU
tracepath -n 10.20.0.5
# tracepath discovers MTU at each hop
# Shows "pmtu" values decreasing at bottleneck hops

# Step 3: Test if ICMP is reaching you
# On the destination host, send ICMP Fragmentation Needed:
# Or just capture: is the ICMP coming back?
tcpdump -i eth0 -n 'icmp and host 10.20.0.5' &
ping -M do -s 1472 10.20.0.5
# If ping fails and NO ICMP captured: black hole (ICMP being dropped)
# If ping fails and ICMP captured: path MTU is smaller than 1472+28=1500

# Step 4: Verify TCP behavior
# Start a file transfer and capture:
tcpdump -i eth0 -n 'tcp and host 10.20.0.5' -v &
scp largefile.txt user@10.20.0.5:/tmp/
# Look for: large TCP segments that are sent but never acknowledged
# If TCP keeps retransmitting the same large segment: black hole
```

## Check for ICMP Blocking (Root Cause)

```bash
# Check if your firewall is blocking ICMP Fragmentation Needed:
iptables -L INPUT -n | grep -E "icmp|ICMP"
iptables -L FORWARD -n | grep -E "icmp|ICMP"

# If you see rules dropping ICMP without allowing type 3:
# Fix: allow ICMP Fragmentation Needed specifically:
iptables -I INPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I OUTPUT -p icmp --icmp-type fragmentation-needed -j ACCEPT
iptables -I FORWARD -p icmp --icmp-type fragmentation-needed -j ACCEPT

# For nftables:
nft add rule inet filter input icmp type destination-unreachable accept
nft add rule inet filter forward icmp type destination-unreachable accept
```

## Fix: TCP MSS Clamping

```bash
# If you can't fix the ICMP blocking (e.g., upstream firewall not yours):
# Use MSS clamping to prevent TCP from using large segments

# Clamp TCP MSS to safe value for all traffic:
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
# This reads the path MTU and clamps MSS accordingly

# Or clamp to specific value (path MTU - 40 bytes for IP+TCP headers):
# For a path MTU of 1400:
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1360

# For outgoing traffic on this host:
iptables -t mangle -A OUTPUT -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu
```

## Fix: Reduce Interface MTU

```bash
# Reduce MTU on the interface to match the path MTU:
# Find actual path MTU first:
tracepath -n destination | grep "pmtu"

# Set MTU on interface:
ip link set eth0 mtu 1450  # Example: for VPN path with 1450 MTU

# Permanent (in /etc/network/interfaces for Debian):
# iface eth0 inet dhcp
#   mtu 1450

# Permanent (in /etc/sysconfig/network-scripts/ifcfg-eth0 for RHEL):
# MTU=1450
```

## Test Fix is Working

```bash
# After applying fix, test with large data:
ping -M do -s 1200 10.20.0.5  # Should succeed
scp largefile.txt user@10.20.0.5:/tmp/  # Should complete

# Watch PMTUD in action (after fix, ICMP should flow):
tcpdump -i eth0 -n 'icmp[0] = 3 and icmp[1] = 4'
# Should see Fragmentation Needed messages when large packets are sent
# TCP will adjust MSS accordingly
```

## Conclusion

PMTUD failures occur when ICMP Fragmentation Needed messages are blocked between sender and receiver, causing TCP to continue sending packets too large for the path. Diagnose with `ping -M do -s SIZE` to find the bottleneck MTU, and `tracepath` to locate the black hole hop. Fix by allowing ICMP type 3 through firewalls. When the upstream firewall is not yours, use TCP MSS clamping (`--clamp-mss-to-pmtu`) to prevent sending oversized segments. This is one of the most common causes of "works here but not there" networking problems in VPN and cloud environments.
