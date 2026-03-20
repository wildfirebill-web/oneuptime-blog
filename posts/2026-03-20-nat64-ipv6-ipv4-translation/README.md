# How to Understand NAT64 for IPv6-to-IPv4 Translation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT64, IPv6, IPv4, Transition

Description: Learn how NAT64 translates between IPv6 and IPv4 networks, how DNS64 works alongside it, and how to deploy NAT64 on Linux.

## What Is NAT64?

NAT64 (RFC 6146) allows IPv6-only clients to communicate with IPv4-only servers by translating IPv6 packets to IPv4 at the border. It is used during IPv6 transition periods when some services are still IPv4-only.

```text
[IPv6-only client]     [NAT64 gateway]      [IPv4-only server]
  2001:db8::1    ←→    Translates    ←→     203.0.113.1
```

## How NAT64 Works

NAT64 uses a **Well-Known Prefix**: `64:ff9b::/96`

To reach an IPv4 address (e.g., 203.0.113.1), an IPv6 client sends traffic to:

```text
64:ff9b::203.0.113.1
= 64:ff9b::cb00:7101  (hex form of 203.0.113.1)
```

The NAT64 gateway:
1. Receives IPv6 packet destined to `64:ff9b::/96`
2. Extracts the embedded IPv4 address (last 32 bits)
3. Translates to IPv4 and forwards to 203.0.113.1
4. Returns translated reply to the IPv6 client

## DNS64

IPv6-only clients can't directly query IPv4 DNS records. DNS64 synthesizes AAAA (IPv6) records from A (IPv4) records:

```text
Client queries: example.com AAAA
DNS64 server:
  → Queries example.com AAAA → No result
  → Queries example.com A → Returns 203.0.113.1
  → Synthesizes AAAA: 64:ff9b::203.0.113.1
Client receives: 64:ff9b::203.0.113.1
Client sends IPv6 traffic to: 64:ff9b::203.0.113.1 → NAT64 translates
```

## Setting Up NAT64 on Linux with Tayga

```bash
# Install Tayga (stateless NAT64)

apt install tayga

# /etc/tayga.conf
tun-device nat64
ipv4-addr 192.168.255.1
prefix 64:ff9b::/96
dynamic-pool 192.168.255.0/24
data-dir /var/lib/tayga

# Start Tayga
tayga --mktun   # Create TUN device
ip link set nat64 up
ip addr add 192.168.255.1 dev nat64
ip route add 64:ff9b::/96 dev nat64
ip route add 192.168.255.0/24 dev nat64

# Enable IP forwarding
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1

# NAT the translated IPv4 traffic
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Run Tayga
tayga
```

## Setting Up DNS64 with BIND

```bash
# /etc/bind/named.conf.options
options {
    dns64 64:ff9b::/96 {
        clients { any; };
        mapped { any; };
        exclude { 64:ff9b::/96; ::ffff:0:0/96; };
    };
};
```

## Testing NAT64

```bash
# From an IPv6-only client:

# Test DNS64 resolution
dig AAAA google.com @dns64_server_ip
# Should return: 64:ff9b::IPv4_address

# Test connectivity
ping6 64:ff9b::8.8.8.8

# Trace the path
traceroute6 64:ff9b::8.8.8.8
```

## NAT64 vs Other IPv6 Transition Mechanisms

| Mechanism | Use Case | Notes |
|-----------|---------|-------|
| NAT64 + DNS64 | IPv6-only → IPv4 | Common in mobile networks |
| 464XLAT | IPv4 apps on IPv6-only | Client-side CLAT + server-side NAT64 |
| Dual Stack | Both IPv4 and IPv6 | Best long-term solution |
| 6to4/Teredo | IPv6 over IPv4 | Legacy, largely deprecated |

## Mobile Networks and NAT64

Many mobile carriers (especially in markets with IPv4 exhaustion) use NAT64:
- Devices get IPv6-only connectivity
- NAT64 gateway handles IPv4 server access
- DNS64 provides synthesized IPv6 addresses for IPv4 resources

## Key Takeaways

- NAT64 translates IPv6 packets to IPv4 using the `64:ff9b::/96` prefix.
- DNS64 synthesizes AAAA records from A records so IPv6-only clients find IPv4 servers.
- Tayga is a popular open-source NAT64 implementation for Linux.
- NAT64 is widely used in mobile networks and IPv6-only deployments.

**Related Reading:**

- [How to Configure NAT on Linux Using iptables](https://oneuptime.com/blog/post/2026-03-20-nat-linux-iptables/view)
