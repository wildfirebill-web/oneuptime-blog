# How to Configure an IPv6 Router on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Linux, Routing, Networking, Router, sysctl

Description: Configure a Linux system as an IPv6 router by enabling packet forwarding, assigning prefix addresses to interfaces, and setting up basic IPv6 routing between network segments.

## Introduction

Linux can function as a fully capable IPv6 router, forwarding packets between network segments, participating in routing protocols, and sending Router Advertisements to clients. This guide covers the essentials of turning a Linux machine with two or more interfaces into an IPv6 router.

## Prerequisites

- Linux system with two network interfaces (e.g., `eth0` for WAN, `eth1` for LAN)
- An IPv6 prefix delegated by your ISP or allocated statically (e.g., `2001:db8:1::/48`)
- Root access

## Step 1: Enable IPv6 Packet Forwarding

By default, Linux does not forward packets between interfaces. Enable IPv6 forwarding:

```bash
# Enable IPv6 forwarding immediately
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Make it persistent across reboots
echo "net.ipv6.conf.all.forwarding = 1" | sudo tee /etc/sysctl.d/50-ipv6-forward.conf
sudo sysctl -p /etc/sysctl.d/50-ipv6-forward.conf
```

## Step 2: Assign IPv6 Addresses to Interfaces

Assign addresses from your delegated prefix to each interface:

```bash
# Assign a /64 subnet to the LAN interface
sudo ip -6 addr add 2001:db8:1:1::1/64 dev eth1

# The WAN interface typically gets its address from the ISP via DHCPv6 or SLAAC
# To statically assign:
sudo ip -6 addr add 2001:db8:0:1::2/64 dev eth0
```

Make these persistent using systemd-networkd or NetworkManager.

## Step 3: Configure Static Routes (Optional)

For multi-router setups, add static routes:

```bash
# Add a route to a remote IPv6 network via a next-hop router
sudo ip -6 route add 2001:db8:2::/48 via 2001:db8:0:1::1 dev eth0

# Verify the routing table
ip -6 route show
```

## Step 4: Install and Configure radvd for Router Advertisements

Router Advertisements (RA) tell LAN clients about the IPv6 prefix so they can autoconfigure:

```bash
# Install radvd
sudo apt-get install radvd

# Create the radvd configuration
sudo tee /etc/radvd.conf > /dev/null << 'EOF'
# Send Router Advertisements on the LAN interface

interface eth1 {
    AdvSendAdvert on;         # Actively send RAs
    AdvManagedFlag off;       # SLAAC (not DHCPv6 for addresses)
    AdvOtherConfigFlag off;   # No DHCPv6 for other info
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;

    prefix 2001:db8:1:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;     # Clients can autoconfigure from this prefix
        AdvRouterAddr on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };
};
EOF

# Enable and start radvd
sudo systemctl enable --now radvd
```

## Step 5: Configure ip6tables (Firewall)

Allow forwarded traffic between the WAN and LAN:

```bash
# Allow established/related traffic to pass through (stateful inspection)
sudo ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow new connections from LAN to WAN
sudo ip6tables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# Block unsolicited inbound connections from WAN to LAN
sudo ip6tables -A FORWARD -i eth0 -o eth1 -j DROP

# Save the rules
sudo ip6tables-save | sudo tee /etc/ip6tables.rules
```

## Step 6: Verify Routing

Test that the router is forwarding packets correctly:

```bash
# From a LAN client, ping an external IPv6 address
ping6 2606:4700:4700::1111  # Cloudflare DNS

# From the router, check that forwarding is active
sysctl net.ipv6.conf.all.forwarding

# Inspect the routing table
ip -6 route show table main
ip -6 route show table local
```

## Checking Router Advertisement Delivery

```bash
# On a LAN client, verify it received a Router Advertisement
# and assigned itself an address from the prefix
ip -6 addr show scope global

# Check the default gateway was learned from RA
ip -6 route show default
# Expected: default via fe80::... dev eth1 proto ra
```

## Conclusion

Configuring Linux as an IPv6 router involves enabling kernel forwarding, assigning addresses to each segment, running radvd to inform clients, and applying firewall rules to control traffic flow. This setup provides a solid foundation for more advanced IPv6 routing scenarios including prefix delegation, DHCPv6, and dynamic routing protocols such as OSPFv3 or BGP.
