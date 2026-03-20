# How to Configure NAT on MikroTik Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, MikroTik, RouterOS, IPv4

Description: Learn how to configure NAT masquerade, static NAT, and port forwarding on MikroTik RouterOS using Winbox and CLI.

## MikroTik NAT Overview

MikroTik RouterOS manages NAT under **IP → Firewall → NAT**. The two main chains are:

| Chain | Use |
|-------|-----|
| srcnat | Source NAT (outbound — masquerade, SNAT) |
| dstnat | Destination NAT (inbound — port forwarding) |

## Outbound NAT (Masquerade)

This is the most common configuration — all LAN traffic masquerades through the WAN interface:

### Winbox GUI

1. Open **IP → Firewall → NAT**
2. Click **+** to add a rule
3. Set:
   - Chain: `srcnat`
   - Out-Interface: `ether1` (WAN interface)
   - Action: `masquerade`
4. Click **OK**

### CLI (RouterOS terminal)

```routeros
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade comment="Outbound NAT"
```

## Port Forwarding (DNAT)

Forward external port to internal server:

### Winbox GUI

1. **IP → Firewall → NAT → +**
2. Chain: `dstnat`
3. Protocol: `tcp`
4. Dst. Port: `80`
5. In Interface: `ether1` (WAN)
6. Action: `dst-nat`
7. To Addresses: `192.168.1.10`
8. To Ports: `80`

### CLI

```routeros
/ip firewall nat add \
    chain=dstnat \
    in-interface=ether1 \
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    to-ports=80 \
    comment="Web server port forward"
```

## Static NAT (1:1)

```routeros
# DNAT: inbound to public IP → forward to private IP
/ip firewall nat add \
    chain=dstnat \
    dst-address=203.0.113.10 \
    action=dst-nat \
    to-addresses=192.168.1.10

# SNAT: outbound from private IP → appear as public IP
/ip firewall nat add \
    chain=srcnat \
    src-address=192.168.1.10 \
    action=src-nat \
    to-addresses=203.0.113.10
```

## Complete NAT Setup Example

```routeros
# Clear existing NAT rules
/ip firewall nat remove [find]

# Outbound masquerade
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="Masquerade LAN"

# Port forward: HTTP
/ip firewall nat add \
    chain=dstnat \
    in-interface=ether1 \
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    to-ports=80

# Port forward: SSH
/ip firewall nat add \
    chain=dstnat \
    in-interface=ether1 \
    protocol=tcp \
    dst-port=22 \
    action=dst-nat \
    to-addresses=192.168.1.20 \
    to-ports=22
```

## Viewing NAT Rules and Counters

```routeros
# List all NAT rules with hit counters
/ip firewall nat print stats

# Monitor active connections
/ip firewall connection print

# Count connections
/ip firewall connection print count-only
```

## Hairpin NAT on MikroTik

```routeros
# Same as port forward but also applies to LAN interface (ether2)
/ip firewall nat add \
    chain=dstnat \
    in-interface=ether2 \
    protocol=tcp \
    dst-address=203.0.113.1 \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    to-ports=80

# Masquerade for hairpin return traffic
/ip firewall nat add \
    chain=srcnat \
    src-address=192.168.1.0/24 \
    dst-address=192.168.1.10 \
    protocol=tcp \
    dst-port=80 \
    out-interface=ether2 \
    action=masquerade
```

## Key Takeaways

- MikroTik uses `srcnat` for outbound NAT and `dstnat` for port forwarding.
- MASQUERADE action automatically uses the WAN interface IP as source.
- `dst-nat` action rewrites the destination IP/port for port forwarding.
- View hit counters with `/ip firewall nat print stats` to verify rules are matching.

**Related Reading:**

- [How to Configure NAT on Linux Using iptables](https://oneuptime.com/blog/post/2026-03-20-nat-linux-iptables/view)
- [How to Set Up NAT with pfSense](https://oneuptime.com/blog/post/2026-03-20-nat-pfsense-setup/view)
- [How to Configure Hairpin NAT for Internal Access to Public Services](https://oneuptime.com/blog/post/2026-03-20-hairpin-nat-internal-access/view)
