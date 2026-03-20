# How to Configure 1:1 NAT for Server Hosting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, IPv4, Linux, Server Hosting

Description: Learn how to configure 1:1 NAT (one-to-one static NAT) to map a dedicated public IP to an internal server for bidirectional traffic.

## What Is 1:1 NAT?

1:1 NAT creates a **permanent bidirectional mapping** between one public IP address and one private IP address. All traffic to/from the public IP is translated to/from the private IP.

```text
External client → 203.0.113.10:80 → [NAT] → 192.168.1.10:80
Internal server 192.168.1.10:80 → [NAT] → 203.0.113.10:80 → External client
```

**Use cases**: Hosting multiple servers on different public IPs, each mapping to a different internal server.

## Configure 1:1 NAT on Linux with iptables

```bash
# Map public IP 203.0.113.10 ↔ private IP 192.168.1.10

# Enable forwarding

echo 1 > /proc/sys/net/ipv4/ip_forward

# Add secondary IP to WAN interface
ip addr add 203.0.113.10/32 dev eth1

# DNAT: inbound to public IP → forward to private IP
iptables -t nat -A PREROUTING -d 203.0.113.10 \
    -j DNAT --to-destination 192.168.1.10

# SNAT: outbound from private IP → appear as public IP
iptables -t nat -A POSTROUTING -s 192.168.1.10 -o eth1 \
    -j SNAT --to-source 203.0.113.10

# Allow forwarded traffic
iptables -A FORWARD -d 192.168.1.10 -j ACCEPT
iptables -A FORWARD -s 192.168.1.10 -j ACCEPT
```

## Multiple 1:1 NAT Mappings

```bash
# Server 1: 203.0.113.10 → 192.168.1.10
ip addr add 203.0.113.10/32 dev eth1
iptables -t nat -A PREROUTING -d 203.0.113.10 -j DNAT --to-destination 192.168.1.10
iptables -t nat -A POSTROUTING -s 192.168.1.10 -o eth1 -j SNAT --to-source 203.0.113.10

# Server 2: 203.0.113.11 → 192.168.1.11
ip addr add 203.0.113.11/32 dev eth1
iptables -t nat -A PREROUTING -d 203.0.113.11 -j DNAT --to-destination 192.168.1.11
iptables -t nat -A POSTROUTING -s 192.168.1.11 -o eth1 -j SNAT --to-source 203.0.113.11
```

## Configure 1:1 NAT with nftables

```bash
table inet nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        ip daddr 203.0.113.10 dnat to 192.168.1.10
        ip daddr 203.0.113.11 dnat to 192.168.1.11
    }
    chain postrouting {
        type nat hook postrouting priority 100;
        ip saddr 192.168.1.10 oifname "eth1" snat to 203.0.113.10
        ip saddr 192.168.1.11 oifname "eth1" snat to 203.0.113.11
    }
}
```

## Configure 1:1 NAT on Cisco

```cisco
! Static NAT: inside 192.168.1.10 = outside 203.0.113.10
ip nat inside source static 192.168.1.10 203.0.113.10
ip nat inside source static 192.168.1.11 203.0.113.11

interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/1
 ip nat outside
```

## Adding the Public IPs to Your Interface

For 1:1 NAT to work, the router must own the public IPs:

```bash
# Add multiple public IPs to eth1
ip addr add 203.0.113.10/32 dev eth1 label eth1:0
ip addr add 203.0.113.11/32 dev eth1 label eth1:1

# Verify
ip addr show eth1
```

If the ISP routes a /29 block to you, these IPs route to your interface automatically without needing to add them manually.

## Verifying 1:1 NAT

```bash
# From external host:
curl http://203.0.113.10      # Should reach 192.168.1.10
curl http://203.0.113.11      # Should reach 192.168.1.11

# From internal server:
curl https://ifconfig.me       # Should return 203.0.113.10

# Check conntrack
conntrack -L | grep 192.168.1.10
```

## Key Takeaways

- 1:1 NAT requires both DNAT (PREROUTING) and SNAT (POSTROUTING) rules for bidirectional translation.
- Each public IP must be added to the WAN interface or routed by the ISP.
- Multiple servers each get their own public IP with separate 1:1 NAT entries.
- On Cisco, `ip nat inside source static private public` handles 1:1 mapping.

**Related Reading:**

- [How to Configure Static NAT on a Router](https://oneuptime.com/blog/post/2026-03-20-configure-static-nat-router/view)
- [How to Configure Source NAT (SNAT) on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-snat-linux/view)
- [How to Configure Destination NAT (DNAT) on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-dnat-linux/view)
