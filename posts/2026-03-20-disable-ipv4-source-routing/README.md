# How to Disable IPv4 Source Routing for Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Security, IPv4, Source Routing, sysctl, Hardening

Description: Disable IPv4 source routing options (LSRR and SSRR) on Linux to prevent attackers from manipulating packet paths and bypassing network controls.

IPv4 source routing allows the sender to specify the exact route a packet must take through the network. Attackers exploit this to bypass firewalls, redirect traffic, and conduct spoofing attacks. Disabling it is a mandatory hardening step.

## What Is Source Routing?

IPv4 source routing is an IP header option that embeds routing directives:

```
IP Header Options:
  LSRR (Loose Source and Record Route) — packet must pass through
         specified hops but can take any path between them
  SSRR (Strict Source and Record Route) — packet must follow
         the exact specified path

Attack uses:
  - Bypass firewall rules (traffic appears from allowed IP)
  - Redirect return traffic to attacker
  - Network topology enumeration
```

## Check Current Status

```bash
# Check if source routing is currently accepted (0 = disabled, 1 = enabled)
cat /proc/sys/net/ipv4/conf/all/accept_source_route

# Check each interface
for iface in /proc/sys/net/ipv4/conf/*/accept_source_route; do
    echo "$iface: $(cat $iface)"
done
```

## Disable Source Routing via sysctl

```bash
# Disable source routing on all interfaces
sudo sysctl -w net.ipv4.conf.all.accept_source_route=0
sudo sysctl -w net.ipv4.conf.default.accept_source_route=0

# Also disable on specific interfaces explicitly
sudo sysctl -w net.ipv4.conf.eth0.accept_source_route=0
sudo sysctl -w net.ipv4.conf.lo.accept_source_route=0

# Verify
sysctl net.ipv4.conf.all.accept_source_route
# net.ipv4.conf.all.accept_source_route = 0
```

## Make Changes Persistent

```bash
# Add to /etc/sysctl.conf or a drop-in file
sudo tee /etc/sysctl.d/99-disable-source-routing.conf << 'EOF'
# Disable IPv4 source routing (security hardening)
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
EOF

# Apply immediately
sudo sysctl -p /etc/sysctl.d/99-disable-source-routing.conf
```

## Block Source Routed Packets with iptables

For defense-in-depth, also block source-routed packets at the firewall level:

```bash
# Drop packets with LSRR or SSRR options set
# The --option flag in iptables can match IP options

# Using ipv4options extension (if available)
sudo iptables -A INPUT -m ipv4options --lsrr -j DROP
sudo iptables -A INPUT -m ipv4options --ssrr -j DROP

# Alternative: block all packets with IP options (more aggressive)
# This drops packets with ANY IP option, including legitimate ones
# sudo iptables -A INPUT -m ipv4options --any-opt -j DROP
```

## Verify with a Test Packet (Optional)

To test that source routing is blocked, you can craft a source-routed packet:

```bash
# Install hping3
sudo apt install hping3

# Send a packet with loose source routing option
# (this should be silently dropped if protection is enabled)
sudo hping3 --lsrr target-ip -c 3

# Check if target received the packet (should not)
sudo tcpdump -i eth0 -n 'ip[0] & 0xf > 5' -c 5
# "ip[0] & 0xf > 5" catches packets with IP options
```

## Compliance Requirements

Disabling source routing is required by many security frameworks:

```
CIS Benchmark for Linux: Controls 3.2.1
DISA STIG for RHEL: V-72283, V-72285
NIST 800-53: SC-5 (Denial of Service Protection)

Check compliance:
sudo grep -r "accept_source_route" /etc/sysctl.conf /etc/sysctl.d/
```

Disabling IPv4 source routing has zero impact on normal network operations and should be treated as a mandatory baseline for any production Linux system.
