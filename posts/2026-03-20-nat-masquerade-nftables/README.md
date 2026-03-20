# How to Set Up NAT with nftables (Masquerade)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nftables, Linux, NAT, Masquerade, Networking, Routing

Description: Configure masquerade NAT with nftables to allow private network hosts to access the internet through a Linux router.

## Introduction

Masquerade NAT rewrites the source IP of outgoing packets to the router's public IP address, enabling hosts on a private network to reach the internet. It is the standard approach on Linux when the public IP is dynamically assigned. This guide walks through the complete setup using nftables.

## Prerequisites

- Linux machine with two interfaces (e.g., `eth0` for WAN, `eth1` for LAN)
- nftables installed
- Root access

## Enable IP Forwarding

Before NAT can work, the kernel must forward packets between interfaces.

```bash
# Enable IPv4 forwarding immediately
sysctl -w net.ipv4.ip_forward=1

# Make it persistent across reboots
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf
```

## Create the NAT Table and Postrouting Chain

NAT rules in nftables live in a dedicated `nat` table with a `postrouting` chain.

```bash
# Create the nat table (ip family for IPv4 only)
nft add table ip nat

# Create the postrouting chain — must use priority 100 for NAT
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }

# Add masquerade rule for traffic leaving the WAN interface (eth0)
nft add rule ip nat postrouting oif "eth0" masquerade
```

## Allow Forwarded Traffic in the Filter Table

The filter forward chain must allow packets flowing between LAN and WAN.

```bash
# Allow forwarded traffic from LAN to WAN
nft add rule inet filter forward iif "eth1" oif "eth0" ct state new,established,related accept

# Allow return traffic from WAN to LAN
nft add rule inet filter forward iif "eth0" oif "eth1" ct state established,related accept
```

## Full Configuration File

A complete nftables configuration for a masquerade NAT router:

```bash
#!/usr/sbin/nft -f

flush ruleset

# IPv4 NAT table
table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;

        # Masquerade all traffic leaving the WAN interface
        oif "eth0" masquerade
    }
}

# Filter table for forwarding
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        iif lo accept
        ct state established,related accept
        ct state invalid drop

        # Allow SSH
        tcp dport 22 accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;

        # Allow LAN to WAN (new connections + established)
        iif "eth1" oif "eth0" ct state new,established,related accept

        # Allow established return traffic from WAN to LAN
        iif "eth0" oif "eth1" ct state established,related accept
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

## Verify NAT Is Working

```bash
# Check postrouting rules are in place
nft list table ip nat

# From a LAN host, verify internet connectivity
ping 8.8.8.8

# Watch NAT connections
nft list ruleset
conntrack -L
```

## Save and Persist

```bash
nft list ruleset > /etc/nftables.conf
systemctl enable nftables
```

## Conclusion

Masquerade NAT with nftables requires IP forwarding enabled, a `nat` table with a `postrouting` chain, and proper `forward` chain rules. The `masquerade` target dynamically sets the source IP to the outbound interface's address, making it ideal for routers with dynamic public IPs.
