# How to Configure IPv6 on Multi-Homed Hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Multi-homing, Routing, Policy Routing, Source Address Selection, Network

Description: Configure IPv6 routing correctly on hosts with multiple network interfaces, ensuring proper source address selection and policy-based routing for each uplink.

## Introduction

Multi-homed IPv6 hosts have multiple network interfaces, each potentially with different IPv6 prefixes from different routers or ISPs. Without proper configuration, IPv6 traffic may use the wrong source address for a given interface, causing asymmetric routing or packets being rejected by reverse-path filtering at upstream routers.

## Understanding the Problem

```text
Multi-homed host:
  eth0: 2001:db8:a::100/64, gateway fe80::gw-a
  eth1: 2001:db8:b::100/64, gateway fe80::gw-b

Default routing table:
  ::/0 via fe80::gw-a dev eth0  (only one default)

Problem: Traffic arriving on eth1 gets response from fe80::gw-a
→ Asymmetric routing, source address mismatch
```

## Step 1: Check Current Configuration

```bash
# Show all interfaces and addresses

ip -6 addr show

# Show routing table
ip -6 route show

# Show routing rules
ip -6 rule show

# Check which route would be used for each destination
ip -6 route get 2001:db8:external::1
```

## Step 2: Configure Policy-Based Routing

```bash
# Create separate routing tables for each interface

# Table 100 for eth0 (ISP A)
sudo ip -6 route add 2001:db8:a::/64 dev eth0 table 100
sudo ip -6 route add default via fe80::gw-a dev eth0 table 100

# Table 200 for eth1 (ISP B)
sudo ip -6 route add 2001:db8:b::/64 dev eth1 table 200
sudo ip -6 route add default via fe80::gw-b dev eth1 table 200

# Rules: use table based on source address
sudo ip -6 rule add from 2001:db8:a::/64 table 100 priority 100
sudo ip -6 rule add from 2001:db8:b::/64 table 200 priority 200

# Verify rules
ip -6 rule show
```

## Step 3: Configure Source Address Selection

```bash
# /etc/gai.conf controls source address selection
# By default, IPv6 prefers longer matching prefix

# Check current preferences
cat /etc/gai.conf | grep -v "^#\|^$"

# Test source address selection
ip -6 route get 2001:db8:external::1
# Look for "src" in output - that's the selected source address

# Force specific source for a route
sudo ip -6 route add 2001:db8:external::/48 \
    via fe80::gw-a dev eth0 src 2001:db8:a::100

# ECMP routing across both interfaces
sudo ip -6 route add default \
    nexthop via fe80::gw-a dev eth0 weight 1 \
    nexthop via fe80::gw-b dev eth1 weight 1
```

## Step 4: Persistent Configuration with systemd-networkd

```bash
# /etc/systemd/network/eth0.network
cat << 'EOF' | sudo tee /etc/systemd/network/eth0.network
[Match]
Name=eth0

[Network]
IPv6AcceptRA=yes
DHCP=ipv6

[IPv6RoutingPolicyRule]
From=2001:db8:a::/64
Table=100
Priority=100

[Route]
Destination=::/0
Gateway=fe80::gw-a
Table=100
EOF

# /etc/systemd/network/eth1.network
cat << 'EOF' | sudo tee /etc/systemd/network/eth1.network
[Match]
Name=eth1

[Network]
IPv6AcceptRA=yes
DHCP=ipv6

[IPv6RoutingPolicyRule]
From=2001:db8:b::/64
Table=200
Priority=200

[Route]
Destination=::/0
Gateway=fe80::gw-b
Table=200
EOF

sudo systemctl restart systemd-networkd
```

## Step 5: Test Multi-Homed Routing

```bash
#!/bin/bash
# test-multihomed-ipv6.sh

echo "=== Multi-homed IPv6 Test ==="

echo ""
echo "Addresses:"
ip -6 addr show scope global | grep "inet6"

echo ""
echo "Routing rules:"
ip -6 rule show

echo ""
echo "Routing tables:"
for table in 100 200; do
    echo "Table $table:"
    ip -6 route show table $table 2>/dev/null || echo "  (empty)"
done

echo ""
echo "Route selection test:"
# Test with source address from each prefix
for src_prefix in "2001:db8:a::100" "2001:db8:b::100"; do
    dest="2001:db8:external::1"
    route=$(ip -6 route get "$dest" from "$src_prefix" 2>/dev/null)
    echo "  From $src_prefix: $route"
done
```

## Step 6: Handle Multiple Default Routes

```bash
# Linux installs only one default route by default
# When RA arrives on eth1, it may replace the eth0 default

# Fix: use metrics to prefer one interface
sudo ip -6 route add default via fe80::gw-a dev eth0 metric 100
sudo ip -6 route add default via fe80::gw-b dev eth1 metric 200
# eth0 default is preferred (lower metric)

# For active-active, use ECMP (equal-cost multipath)
sudo ip -6 route add default \
    nexthop via fe80::gw-a dev eth0 weight 1 \
    nexthop via fe80::gw-b dev eth1 weight 1

# Verify ECMP
ip -6 route show default
```

## Conclusion

Multi-homed IPv6 hosts require policy-based routing to ensure each interface uses the appropriate default gateway and source address. Configure separate routing tables per interface with `ip -6 route add ... table N` and routing rules with `ip -6 rule add from <prefix> table N`. This ensures that traffic arriving on eth1 is responded to via eth1, and traffic arriving on eth0 responds via eth0. Use `systemd-networkd` for persistent multi-homed IPv6 configuration with `[IPv6RoutingPolicyRule]` sections.
