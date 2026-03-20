# How to Configure IPv6 Stateless Router on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Linux, Stateless, SLAAC, Router, Networking

Description: Configure a Linux system as a fully stateless IPv6 router using SLAAC and Router Advertisements, requiring no DHCPv6 server for client address assignment.

## Introduction

A stateless IPv6 router forwards packets and sends Router Advertisements that enable clients to configure their own addresses via SLAAC, with no centralized address assignment (no DHCPv6). This is the simplest IPv6 routing configuration and is suitable for home networks, small offices, and environments that prefer zero-configuration networking.

## What "Stateless" Means

In a stateless configuration:
- The router does NOT maintain a database of address assignments
- Clients derive their own addresses from the advertised prefix + their IID
- The router sends RAs with M=0 (Managed flag off) and O=0 (Other flag off)
- DNS can be delivered via RDNSS in the RA (RFC 8106)

## Step 1: Enable IPv6 Forwarding

```bash
# Enable IPv6 packet forwarding between interfaces
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Persist across reboots
echo "net.ipv6.conf.all.forwarding = 1" | sudo tee /etc/sysctl.d/50-ipv6-forward.conf
sudo sysctl -p /etc/sysctl.d/50-ipv6-forward.conf
```

## Step 2: Assign Addresses to Router Interfaces

```bash
# WAN interface - typically gets address via SLAAC from ISP or DHCPv6
# For this example, assume it's configured via DHCPv6 prefix delegation
# and we have 2001:db8::/48 delegated to us

# LAN interface - assign the first /64 from the delegated prefix
sudo ip -6 addr add 2001:db8:1:1::1/64 dev eth1

# Verify
ip -6 addr show eth1
```

## Step 3: Install and Configure radvd

```bash
sudo apt-get install radvd

sudo tee /etc/radvd.conf > /dev/null << 'EOF'
# Stateless router configuration
# M=0: No DHCPv6 for addresses
# O=0: No DHCPv6 for other configuration

interface eth1 {
    AdvSendAdvert on;

    # M flag off: clients use SLAAC, not DHCPv6
    AdvManagedFlag off;

    # O flag off: no DHCPv6 for DNS either (use RDNSS below)
    AdvOtherConfigFlag off;

    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;
    AdvDefaultLifetime 1800;

    prefix 2001:db8:1:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };

    # Deliver DNS via RA (no DHCPv6 needed)
    RDNSS 2001:db8:1:1::53 2606:4700:4700::1111 {
        AdvRDNSSLifetime 600;
    };

    DNSSL example.com {
        AdvDNSSLLifetime 600;
    };
};
EOF

sudo systemctl enable --now radvd
```

## Step 4: Configure Basic Firewall

Allow IPv6 forwarding with a stateful firewall:

```bash
# Allow established and related traffic
sudo ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow LAN to WAN traffic
sudo ip6tables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# Drop unsolicited inbound from WAN
sudo ip6tables -A FORWARD -i eth0 -o eth1 -j DROP

# Allow all LAN-local traffic
sudo ip6tables -A INPUT -i eth1 -j ACCEPT

# Persist rules
sudo ip6tables-save | sudo tee /etc/ip6tables.rules
```

## Step 5: Verify the Stateless Configuration

```bash
# Verify no DHCPv6 server is running
sudo ss -6 -u -l | grep 547
# Should return no output (port 547 = DHCPv6 server)

# From a client, verify SLAAC address was obtained
ip -6 addr show scope global
# Address should be formed from the prefix 2001:db8:1:1::/64

# Verify no M or O flag in received RAs
rdisc6 eth0
# Should show:
# Stateful address conf. : No   (M=0)
# Stateful other conf.   : No   (O=0)
```

## Optional: Add an Internal IPv6 DNS Resolver

For DNS to work in RDNSS, you need a resolver that listens on the advertised address:

```bash
# Use systemd-resolved as the internal resolver
sudo systemctl enable --now systemd-resolved

# Or configure bind9 to listen on the IPv6 address
# /etc/bind/named.conf.options:
# listen-on-v6 { 2001:db8:1:1::53; };

# Update radvd.conf to point to the actual resolver IP
```

## Conclusion

A stateless IPv6 router on Linux is straightforward to configure: enable forwarding, assign LAN addresses, run radvd with M=0/O=0, and deliver DNS via RDNSS. The absence of DHCPv6 reduces operational complexity and eliminates a potential single point of failure. This configuration is ideal for environments where clients support SLAAC and RDNSS-based DNS configuration, which includes all modern operating systems.
