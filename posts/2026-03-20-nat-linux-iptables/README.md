# How to Configure NAT on Linux Using iptables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, Linux, iptables, IPv4

Description: Learn how to configure NAT on Linux using iptables, including MASQUERADE, SNAT, DNAT, and port forwarding rules.

## NAT Tables in iptables

iptables uses the `nat` table for NAT rules:

| Chain | Use |
|-------|-----|
| PREROUTING | Modify destination before routing (DNAT, port forwarding) |
| OUTPUT | Modify locally-generated packets |
| POSTROUTING | Modify source after routing (SNAT, MASQUERADE) |

## Prerequisites

```bash
# Enable IP forwarding

echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# Identify interfaces
# eth0 = inside (LAN)  192.168.1.1/24
# eth1 = outside (WAN) 203.0.113.1/24
```

## Outbound NAT: MASQUERADE (Dynamic Public IP)

Use MASQUERADE when the public IP is assigned by DHCP:

```bash
# Allow all LAN traffic to internet (automatically uses eth1's current IP)
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 -j MASQUERADE

# Allow forwarding
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Outbound NAT: SNAT (Static Public IP)

Preferred when you have a static public IP:

```bash
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 \
    -j SNAT --to-source 203.0.113.1

iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Inbound NAT: DNAT (Port Forwarding)

```bash
# Forward port 80 to internal web server
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.10:80

# Forward external 2222 → internal SSH 22
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 2222 \
    -j DNAT --to-destination 192.168.1.20:22

# Allow those forwarded connections
iptables -A FORWARD -i eth1 -p tcp -d 192.168.1.10 --dport 80 -j ACCEPT
iptables -A FORWARD -i eth1 -p tcp -d 192.168.1.20 --dport 22 -j ACCEPT
```

## Static 1:1 NAT

```bash
# Bidirectional 1:1 NAT: 192.168.1.10 ↔ 203.0.113.10
iptables -t nat -A PREROUTING -i eth1 -d 203.0.113.10 \
    -j DNAT --to-destination 192.168.1.10

iptables -t nat -A POSTROUTING -o eth1 -s 192.168.1.10 \
    -j SNAT --to-source 203.0.113.10
```

## Viewing NAT Rules

```bash
# Show all NAT rules
iptables -t nat -L -n -v

# Show with line numbers (for deletion)
iptables -t nat -L -n -v --line-numbers

# Show active translations (conntrack)
conntrack -L 2>/dev/null | head -20
```

## Deleting NAT Rules

```bash
# Delete by rule number
iptables -t nat -D POSTROUTING 1

# Delete by matching the rule
iptables -t nat -D POSTROUTING -s 192.168.1.0/24 -o eth1 -j MASQUERADE
```

## Saving and Restoring Rules

```bash
# Save current rules
iptables-save > /etc/iptables/rules.v4

# Restore
iptables-restore < /etc/iptables/rules.v4

# Auto-restore on boot (Debian/Ubuntu)
apt install iptables-persistent
netfilter-persistent save
```

## Complete NAT Gateway Configuration

```bash
#!/bin/bash
# Complete NAT gateway setup

# Enable forwarding
sysctl -w net.ipv4.ip_forward=1

# Flush existing NAT rules
iptables -t nat -F

# Outbound NAT
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 -j MASQUERADE

# Port forwarding
iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.10:80

# Forwarding rules
iptables -P FORWARD DROP
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -p tcp -d 192.168.1.10 --dport 80 -j ACCEPT
```

## Key Takeaways

- Use MASQUERADE for dynamic IPs; use SNAT for static public IPs.
- DNAT in PREROUTING handles port forwarding; add FORWARD rules too.
- `conntrack -L` shows active NAT sessions.
- Persist rules with `iptables-save` and `iptables-persistent`.

**Related Reading:**

- [How to Configure NAT on Linux Using nftables](https://oneuptime.com/blog/post/2026-03-20-nat-linux-nftables/view)
- [How to Configure Source NAT (SNAT) on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-snat-linux/view)
- [How to Configure Destination NAT (DNAT) on Linux](https://oneuptime.com/blog/post/2026-03-20-configure-dnat-linux/view)
