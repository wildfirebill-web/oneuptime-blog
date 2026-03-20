# How to Secure IPv6 on Home Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Security, Home Network, Firewall, NDP, Privacy

Description: Secure your home IPv6 network by configuring firewalls, enabling privacy extensions, preventing rogue RAs, and hardening NDP.

## Why IPv6 Security Differs from IPv4

In IPv4, NAT provides implicit security by hiding internal devices behind a single public IP. In IPv6, every device has a globally routable address — making explicit firewall rules essential.

## 1. Enable IPv6 Firewall on Your Router

Most modern routers block all unsolicited inbound IPv6 connections by default. Verify this is active:

- **UniFi**: Settings → Firewall → IPv6 Rules → Default deny WAN In
- **OpenWRT**: Network → Firewall → IPv6 rule set — verify "input" policy is "reject"
- **Asus**: Advanced → IPv6 → Firewall: Enable

If your router doesn't have an explicit IPv6 firewall, apply it at the OS level on each device using nftables or ip6tables.

## 2. Apply nftables Firewall on Linux Devices

For individual Linux devices (servers, Raspberry Pi, etc.) on your network:

```bash
# /etc/nftables.conf - secure IPv6 for a home server

table ip6 filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Allow loopback
        iif "lo" accept

        # Allow established connections
        ct state established,related accept

        # Allow ICMPv6 (REQUIRED for IPv6 to function)
        ip6 nexthdr icmpv6 accept

        # Allow SSH from local LAN only
        ip6 saddr 2001:db8:home::/64 tcp dport 22 accept

        # Allow HTTPS from anywhere (if hosting a web server)
        tcp dport 443 accept

        # Log and drop everything else
        log prefix "IPv6-DROP: " drop
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }
}
```

Apply with: `nft -f /etc/nftables.conf`

## 3. Enable Privacy Extensions

By default, SLAAC generates IPv6 addresses from the device's MAC address, which can track your device across networks. Enable privacy extensions to use temporary random addresses:

**Linux:**
```bash
# Enable privacy extensions permanently
echo "net.ipv6.conf.all.use_tempaddr=2" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.use_tempaddr=2" >> /etc/sysctl.conf
sysctl -p
```

**Windows:** Privacy extensions are enabled by default.

**macOS:** Privacy extensions are enabled by default.

## 4. Prevent Rogue Router Advertisements

Malicious devices can send fake RAs to redirect IPv6 traffic. Prevent this on your router:

**OpenWRT - radvd only on trusted interfaces:**
```
# /etc/config/radvd
config interface
    option interface 'br-lan'
    option IgnoreIfMissing '0'
    # Only send RA from the router, not from other devices
```

Apply RA Guard on managed switches (if available):

```bash
# On Linux bridge: filter RAs from non-router sources
ebtables -A INPUT -p IPv6 --ip6-proto 58 --ip6-icmp-type 134 \
  -i eth1 -j DROP    # Block RAs from non-router port
```

## 5. Secure ICMPv6

ICMPv6 is essential for IPv6 operation, but only specific types need to be allowed. Apply type-specific filters:

```bash
# Allow required ICMPv6 types, block others
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type echo-request -j ACCEPT    # Ping
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type echo-reply -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 1 -j ACCEPT   # Dest unreachable
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 2 -j ACCEPT   # Packet Too Big
ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 133-136 -j ACCEPT  # NDP
ip6tables -A INPUT -p ipv6-icmp -j DROP    # Block all other ICMPv6
```

## 6. Monitor IPv6 Connections

Use netstat or ss to monitor active IPv6 connections:

```bash
# List active IPv6 connections
ss -6 -tuln

# Show IPv6 neighbor cache (who's on your network)
ip -6 neigh show

# Monitor in real-time
watch -n 5 "ip -6 neigh show | grep -v FAILED"
```

## Conclusion

Securing IPv6 on home networks requires explicit firewall rules (since NAT is gone), privacy extensions for address anonymization, and RA guard to prevent rogue advertisement attacks. Unlike IPv4 where NAT provided accidental security, IPv6 demands intentional, layered security at both the router and device level.
