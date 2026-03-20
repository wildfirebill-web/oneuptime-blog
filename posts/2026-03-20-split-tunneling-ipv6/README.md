# How to Configure Split Tunneling for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, VPN, Split Tunneling, Routing, WireGuard, OpenVPN

Description: A guide to configuring IPv6 split tunneling in WireGuard and OpenVPN, routing specific IPv6 prefixes through the VPN while allowing other IPv6 traffic to go direct.

IPv6 split tunneling routes only specific IPv6 prefixes through the VPN tunnel while allowing other IPv6 traffic to flow directly to the Internet. This is useful when you need access to internal IPv6 resources through the VPN without routing all IPv6 through it.

## Split Tunneling Concept

```
Internal IPv6 traffic → VPN tunnel → Internal network
Public IPv6 traffic   → Direct to Internet (not through VPN)
```

## WireGuard IPv6 Split Tunnel

In WireGuard, `AllowedIPs` defines what goes through the tunnel:

```ini
# /etc/wireguard/wg0.conf (split tunnel)

[Interface]
Address = 10.0.0.2/32
Address = fd00:wg::2/128
PrivateKey = <private-key>

# No DNS override — use system DNS for public names

[Peer]
PublicKey = <server-public-key>
Endpoint = vpn.example.com:51820

# Only route internal IPv6 prefixes through VPN
AllowedIPs = 10.0.0.0/8,              # Internal IPv4
             fd00:internal::/48,        # Internal IPv6 unique-local
             2001:db8:office::/48,      # Office IPv6 public prefix

# Public IPv6 (::/0) is NOT included — goes direct
```

## Calculating Split Tunnel AllowedIPs

To route "everything except certain subnets":

```bash
# Install wireguard-tools for allowed-ips calculator
# Or use the online calculator at https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/

# Route everything EXCEPT your local network
# Instead of ::/0, use:
# 2000::/3  — covers all global unicast (public IPv6)
# (excludes fc00::/7 — unique local, link-local, etc.)

AllowedIPs = 2000::/3    # Only global IPv6 through VPN
```

## OpenVPN IPv6 Split Tunnel

```ini
# /etc/openvpn/client.conf

client
dev tun
tun-ipv6
remote vpn.example.com 1194 udp

ca ca.crt
cert client.crt
key client.key

# Do NOT use redirect-gateway for IPv6
# Instead, specify exact prefixes to route through VPN
route-ipv6 2001:db8:office::/48
route-ipv6 fd00:internal::/48

pull
```

Server configuration for per-client routes:

```ini
# /etc/openvpn/server.conf
# Push specific IPv6 routes to clients
push "route-ipv6 2001:db8:office::/48"
push "route-ipv6 fd00:internal::/48"

# Do NOT push: "route-ipv6 ::/0"   (would be full tunnel)
```

## Routing IPv6 Based on Domain

For more sophisticated split tunneling based on domain names:

```bash
# Use DNS-based routing with systemd-resolved or dnsmasq
# Route DNS queries for internal domains to internal DNS
# Those domains return internal IPv6 addresses → routes through VPN

# /etc/systemd/resolved.conf.d/vpn-split.conf
[Resolve]
# Use internal DNS for internal domain
DNS=fd00:internal::53
Domains=~internal.example.com ~corp.example.com
```

## Testing Split Tunnel Configuration

```bash
# Verify internal IPv6 goes through VPN
traceroute6 fd00:internal::server
# First hop should be VPN gateway

# Verify public IPv6 goes direct
traceroute6 2001:4860:4860::8888
# First hop should be your local gateway (not VPN)

# Check routing table shows split routes
ip -6 route show | grep wg0    # VPN routes
ip -6 route show | grep eth0   # Direct routes
```

## Dynamic Split Tunnel with Route Injection

```bash
# Script to add/remove IPv6 split tunnel routes when VPN connects
# /etc/wireguard/wg-up.sh
#!/bin/bash

# Add internal IPv6 routes when VPN comes up
ip -6 route add 2001:db8:office::/48 dev wg0
ip -6 route add fd00:internal::/48 dev wg0

# wg-quick up hook:
# PostUp = /etc/wireguard/wg-up.sh
# PreDown = /etc/wireguard/wg-down.sh
```

## Security Considerations for IPv6 Split Tunneling

| Risk | Mitigation |
|---|---|
| Internal resource exposure | Firewall controls on VPN server |
| DNS leaks for internal names | Use internal DNS only for internal domains |
| IPv6 direct routing bypasses monitoring | Log direct IPv6 separately |
| Misconfiguration allows full bypass | Test all routes after configuration |

Split tunneling reduces VPN load and improves performance for public content while keeping internal IPv6 resources accessible through the secure tunnel — the key is accurately defining which prefixes should be routed through the VPN.
