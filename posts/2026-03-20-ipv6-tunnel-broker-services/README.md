# How to Understand IPv6 Tunnel Broker Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Tunnel Broker, 6in4, Connectivity, ISP

Description: Learn what IPv6 tunnel broker services are, how they provide IPv6 connectivity over IPv4 networks, and which providers offer tunnel broker services.

## Overview

An IPv6 tunnel broker is a service that provides IPv6 connectivity to users or organizations that do not have native IPv6 from their ISP. The broker operates IPv6 servers with global IPv6 routing and establishes a 6in4 (SIT, protocol 41) tunnel to the customer. The customer's traffic exits through the broker into the native IPv6 internet.

## How Tunnel Brokers Work

```mermaid
graph LR
    A[Your Host<br/>IPv4: 203.0.113.10<br/>No native IPv6] -->|6in4 proto 41| B[Tunnel Broker PoP<br/>IPv4: 198.51.100.1<br/>IPv6: 2001:db8::1]
    B --> C[IPv6 Internet]
    C --> D[IPv6 Servers]

    B -->|Assigns to you| E[IPv6 /64 for tunnel endpoint<br/>IPv6 /48 for your LAN]
```

When you sign up:
1. Broker assigns you a /64 prefix for the tunnel link (e.g., `2001:db8:abcd::/64`)
2. Broker assigns you a /48 prefix for your LAN (e.g., `2001:db8:abcd::/48`)
3. You configure the SIT tunnel to the broker's endpoint
4. All your IPv6 traffic exits via the broker

## Tunnel Broker Providers

| Provider | URL | Tunnel Types | IPv6 Prefix | Free? |
|---|---|---|---|---|
| Hurricane Electric (HE) | tunnelbroker.net | 6in4 | /64 + /48 | Yes |
| SixXS | sixXS.net | 6in4, AYIYA | /64 + /48 | Was free — shut down 2017 |
| NetAssist | net.ua/ipv6 | 6in4 | /64 + /48 | Yes (Ukraine) |
| Freenet6 / Gogo6 | defunct | 6in4 | — | Closed |

**Hurricane Electric** is the dominant tunnel broker today, operating PoPs in:
- North America: Fremont, Los Angeles, Dallas, Chicago, New York, Miami
- Europe: Amsterdam, Frankfurt, London, Stockholm, Paris
- Asia: Hong Kong, Tokyo, Singapore

## What a Tunnel Broker Provides

When you register:

```
Tunnel details page shows:
  Server IPv4 Address:  198.51.100.1        (HE's endpoint)
  Server IPv6 Address:  2001:db8:abcd::1/64 (HE's tunnel IP)
  Client IPv4 Address:  203.0.113.10        (your WAN IPv4 — auto-detected)
  Client IPv6 Address:  2001:db8:abcd::2/64 (your tunnel IP)

Routed IPv6 Prefix:
  2001:db8:abcd::/48    (48 /64 subnets for your LAN)
```

## Configuration Examples Provided by Brokers

Hurricane Electric provides ready-to-use configuration for multiple platforms:

```bash
# Linux example (from HE's tunnel configuration page)
modprobe ipv6
ip tunnel add he-ipv6 mode sit remote 198.51.100.1 local 203.0.113.10 ttl 255
ip link set he-ipv6 up
ip addr add 2001:db8:abcd::2/64 dev he-ipv6
ip route add ::/0 dev he-ipv6
ip -f inet6 addr

# Cisco IOS example (from HE's tunnel configuration page)
ipv6 unicast-routing
interface Tunnel0
  ipv6 address 2001:db8:abcd::2/64
  tunnel source GigabitEthernet0/0
  tunnel mode ipv6ip
  tunnel destination 198.51.100.1
ipv6 route ::/0 Tunnel0
```

## Selecting a Tunnel PoP

Choose a PoP close to your location for lowest latency:

```bash
# Ping multiple HE PoPs to find lowest latency
ping 216.218.218.218   # HE Fremont, CA
ping 216.66.80.30      # HE Dallas, TX
ping 184.105.253.10    # HE New York, NY

# HE also provides latency info on tunnelbroker.net
# Create up to 5 free tunnels on HE
```

## Dynamic IPv4 Update

If your ISP uses dynamic IPv4, the broker needs to know your current IP:

```bash
# Update tunnel endpoint when IPv4 changes
# Hurricane Electric provides an update URL:
curl -4 -s "https://USERNAME:APIKEY@ipv4.tunnelbroker.net/nic/update?hostname=TUNNELID"

# Add to dhclient-exit-hooks for auto-update on DHCP lease:
# /etc/dhcp/dhclient-exit-hooks.d/update-he-tunnel
if [ "$reason" = "BOUND" ] || [ "$reason" = "RENEW" ]; then
    NEW_IP=$new_ip_address
    curl -4 -s "https://USER:KEY@ipv4.tunnelbroker.net/nic/update?hostname=TUNNELID&myip=$NEW_IP"
    ip tunnel change he-ipv6 local $NEW_IP
fi
```

## Security Considerations

Using a tunnel broker means:
- All IPv6 traffic passes through the broker's infrastructure
- Broker can see plaintext IPv6 traffic (use TLS/IPsec for sensitive apps)
- Tunnel broker PoP becomes a dependency for IPv6 availability
- Protocol 41 must be allowed outbound from your firewall

```bash
# Allow only broker's IPv4 for protocol 41
iptables -A INPUT  -p 41 -s 198.51.100.1 -j ACCEPT
iptables -A INPUT  -p 41 -j DROP
iptables -A OUTPUT -p 41 -d 198.51.100.1 -j ACCEPT
iptables -A OUTPUT -p 41 -j DROP
```

## When to Use a Tunnel Broker

Use a tunnel broker when:
- Your ISP does not provide IPv6 (e.g., older DSL/cable provider)
- You need IPv6 for development or testing
- You want to host IPv6-accessible services temporarily

Do not use in production when:
- Native IPv6 is available (always prefer native)
- Low latency is critical (tunnel adds overhead and HE hop)
- High-security environment (all traffic through third party)

## Summary

IPv6 tunnel brokers provide free IPv6 connectivity over IPv4 by establishing 6in4 tunnels. Hurricane Electric (tunnelbroker.net) is the primary provider today. They assign a /64 for the tunnel link and a /48 for your LAN. Configuration is straightforward: `ip tunnel add mode sit remote <broker-ip> local <your-ip>`. Tunnel brokers are a valid solution when native IPv6 is unavailable but should be replaced with native dual-stack when the ISP upgrades. Restrict protocol 41 to the broker's IPv4 address only at your firewall.
