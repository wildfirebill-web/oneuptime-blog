# How to Detect If You Are Behind Carrier-Grade NAT (CGNAT)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, NAT, CGNAT, IPv4, ISP

Description: Learn how to detect if your ISP is placing you behind Carrier-Grade NAT and what it means for port forwarding and hosting services.

## What Is CGNAT?

Carrier-Grade NAT (CGNAT), also called Large-Scale NAT (LSN), is NAT performed by your ISP before traffic reaches your home router. Your ISP assigns you a **shared private IP** from the 100.64.0.0/10 range (RFC 6598) instead of a public IP.

```
[Your PC]          [Your Router]    [ISP CGNAT]       [Internet]
192.168.1.10 → NAT → 100.64.x.x → NAT → 203.0.113.1 → 8.8.8.8
```

## CGNAT Address Range

The shared address space defined in RFC 6598:

```
100.64.0.0/10 (100.64.0.0 – 100.127.255.255)
```

This range is specifically reserved for ISP CGNAT use and is not routable on the internet.

## Method 1: Check Your Router's WAN IP

```bash
# Check if your router's WAN IP is in 100.64.0.0/10
# From inside your network, check the gateway IP assigned by ISP

import ipaddress

def is_cgnat(ip_str):
    ip = ipaddress.ip_address(ip_str)
    cgnat = ipaddress.ip_network('100.64.0.0/10')
    private_ranges = [
        ipaddress.ip_network('10.0.0.0/8'),
        ipaddress.ip_network('172.16.0.0/12'),
        ipaddress.ip_network('192.168.0.0/16'),
    ]
    if ip in cgnat:
        return "CGNAT (100.64.0.0/10)"
    for r in private_ranges:
        if ip in r:
            return f"Private ({r})"
    return "Public IP"

# Check your router's WAN IP
wan_ip = "100.64.0.1"  # example
print(f"{wan_ip}: {is_cgnat(wan_ip)}")
```

## Method 2: Compare Router WAN IP to External IP

```bash
# Get your router's WAN IP (varies by router)
# Many routers expose this at 192.168.1.1 under WAN settings

# Get external IP (what the internet sees)
curl -s https://ifconfig.me

# If router WAN IP ≠ external IP: you're behind NAT (CGNAT or double NAT)
# If router WAN IP = external IP: you have a public IP
```

## Method 3: traceroute Analysis

```bash
# Run traceroute to an internet host
traceroute 8.8.8.8

# Indicators of CGNAT:
# - First hop: your router (192.168.1.1)
# - Second hop: 100.64.x.x (CGNAT shared space)
# - Third+ hop: ISP infrastructure

# Example CGNAT traceroute:
# 1. 192.168.1.1
# 2. 100.64.1.1     ← CGNAT device
# 3. 203.0.113.1    ← ISP public IP
```

## Method 4: Try Port Forwarding Test

```bash
# Set up a listener on your machine
python3 -m http.server 8080

# Configure port forward on your router: WAN:8080 → LAN:8080
# Then test from outside (use your phone on mobile data)
curl http://YOUR_EXTERNAL_IP:8080

# If you can't reach it even with correct port forwarding, you're likely behind CGNAT
```

## What CGNAT Prevents

- Hosting public servers (web, game, VPN)
- Port forwarding that works from outside the ISP network
- Getting a consistent public IP for DNS
- Running a personal VPN server

## Workarounds for CGNAT

1. **Request a public IP from ISP** — many offer it as an upgrade
2. **Use a VPN with port forwarding** — e.g., Mullvad, AirVPN offer port forwarding through their servers
3. **Use a cloud relay** — route traffic through a VPS (frp, ngrok, bore)
4. **IPv6** — IPv6 bypasses CGNAT entirely (each device gets a global IPv6)

## Key Takeaways

- CGNAT uses the 100.64.0.0/10 range (RFC 6598 shared address space).
- If your router's WAN IP is in 100.64.0.0/10 or differs from your external IP, you're behind CGNAT.
- traceroute showing 100.64.x.x as an early hop confirms CGNAT.
- Contact your ISP for a public IP, or use cloud relay/IPv6 as workarounds.

**Related Reading:**

- [How to Work Around CGNAT for Port Forwarding](https://oneuptime.com/blog/post/2026-03-20-cgnat-workaround-port-forwarding/view)
- [How to Understand the Shared Address Space (100.64.0.0/10)](https://oneuptime.com/blog/post/2026-03-20-shared-address-space-100-64/view)
- [How to Diagnose and Fix Double NAT Problems](https://oneuptime.com/blog/post/2026-03-20-double-nat-problems/view)
