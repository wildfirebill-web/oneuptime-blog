# How to Configure IPv6 Firewall on Home Routers - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Firewall, Home Router, Security, Ip6tables, OpenWrt

Description: Configure IPv6 firewall rules on home routers to protect devices from inbound internet threats, allowing stateful return traffic while blocking unsolicited connections.

## Why IPv6 Firewall Matters

With IPv6, devices have globally routable addresses - there is no NAT to hide them. Every device is directly reachable from the internet. A firewall is essential.

```text
IPv4 with NAT:
  Internet → Router NAT → All home devices hidden

IPv6 without NAT:
  Internet → Router → Each device has public IPv6 address
  → Firewall rules REQUIRED to block unsolicited inbound
```

## OpenWrt IPv6 Firewall (ip6tables)

OpenWrt uses nftables/ip6tables for IPv6 packet filtering.

```bash
# View current IPv6 firewall rules

ip6tables -L -n -v --line-numbers

# Default OpenWrt IPv6 firewall zones:
#   WAN zone: reject inbound, accept outbound
#   LAN zone: accept all

# Basic stateful firewall - allow established, block new inbound
ip6tables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
ip6tables -A INPUT -m state --state NEW -i eth0 -j DROP
ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
ip6tables -A FORWARD -m state --state NEW -i eth0 -j DROP

# Always allow ICMPv6 (required for IPv6 to function)
ip6tables -A INPUT -p ipv6-icmp -j ACCEPT
ip6tables -A FORWARD -p ipv6-icmp -j ACCEPT
```

## Essential ICMPv6 Rules

ICMPv6 is mandatory for IPv6 to work - do not block it all.

```bash
# ICMPv6 types that MUST be allowed (RFC 4890)

# Allow Neighbor Discovery Protocol
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 133 -j ACCEPT   # RS
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 134 -j ACCEPT   # RA
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 135 -j ACCEPT   # NS
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 136 -j ACCEPT   # NA

# Allow Path MTU Discovery
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 2 -j ACCEPT     # Packet Too Big

# Allow Ping (optional but useful)
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 128 -j ACCEPT   # Echo Request
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 129 -j ACCEPT   # Echo Reply

# Block all other ICMPv6 from WAN
ip6tables -A INPUT -p ipv6-icmp -i eth0 -j DROP
```

## OpenWrt Firewall Configuration (UCI)

Manage IPv6 firewall via OpenWrt's Unified Configuration Interface.

```bash
# View current firewall config
cat /etc/config/firewall

# Add rule to allow inbound SSH to specific home server
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-SSH-inbound-IPv6'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest='lan'
uci set firewall.@rule[-1].dest_ip='2001:db8:home:1::server'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='22'
uci set firewall.@rule[-1].target='ACCEPT'
uci set firewall.@rule[-1].family='ipv6'
uci commit firewall
/etc/init.d/firewall restart

# Verify rule was added
ip6tables -L FORWARD -n -v | grep "22"
```

## nftables IPv6 Firewall (Modern Linux Routers)

Many modern routers use nftables instead of ip6tables.

```bash
# /etc/nftables.conf - IPv6 home router firewall

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Allow established connections
        ct state established,related accept

        # Allow loopback
        iif lo accept

        # Allow all ICMPv6 (required)
        ip6 nexthdr icmpv6 accept

        # Allow SSH from LAN only
        iif "br-lan" tcp dport 22 accept

        # Allow DHCPv6 client replies from ISP
        ip6 nexthdr udp udp dport 546 accept

        # Drop everything else
        drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;

        # Allow established connections (stateful)
        ct state established,related accept

        # Allow ICMPv6 through (NDP, PMTU, etc.)
        ip6 nexthdr icmpv6 accept

        # Allow LAN to WAN (outbound)
        iif "br-lan" oif "eth0" accept

        # Drop new inbound from WAN
        iif "eth0" drop
    }
}
```

## Test Firewall Effectiveness

```bash
# From an external IPv6 host (or phone on LTE with IPv6)

# Test that random ports are blocked
nc -6 -v 2001:db8:home:1::server 80   # Should timeout/refused
nc -6 -v 2001:db8:home:1::server 443  # Should timeout/refused (if not server)

# Test that allowed services work
nc -6 -v 2001:db8:home:1::server 22   # Should connect (if SSH allowed)

# From home device - verify outbound still works
ping6 2606:4700:4700::1111    # Should work (stateful allows return)
curl -6 https://google.com    # Should work

# Check if NDP/RA still works (ICMPv6 not blocked)
rdisc6 eth0    # Should receive RA from router
```

## Conclusion

IPv6 firewall on home routers must block all unsolicited inbound connections while allowing stateful return traffic for outbound sessions. Always permit all ICMPv6 traffic as it is essential for Neighbor Discovery, Path MTU Discovery, and Router Advertisements. On OpenWrt, use the UCI firewall interface to add specific inbound allow rules for services you intend to expose (SSH, HTTPS). Use nftables for modern routers - define forward chain policy as `drop` and add explicit accept rules for LAN→WAN traffic. Test firewall effectiveness by scanning your public IPv6 addresses from an external connection on mobile data.
