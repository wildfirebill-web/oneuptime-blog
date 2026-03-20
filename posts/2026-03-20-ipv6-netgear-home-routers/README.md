# How to Configure IPv6 on Netgear Home Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Netgear, Home Router, Orbi, DHCPv6

Description: Enable IPv6 on Netgear Nighthawk and Orbi routers, configure DHCPv6-PD from your ISP, and set up SLAAC for home devices.

## Supported Netgear Models

IPv6 is supported on Nighthawk (R7000, R8000, RAX series) and Orbi mesh systems. Access settings at `routerlogin.net` or `192.168.1.1`.

## GUI Configuration (Nighthawk)

```
Path: Advanced → Advanced Setup → IPv6

Internet Connection Type:
  Auto Detect (let router choose — recommended)
  or
  DHCPv6        — for most ISPs with DHCPv6-PD
  Auto Config   — for SLAAC-only ISPs
  6to4 Tunnel   — legacy tunnel (avoid)
  Fixed         — static IPv6 address

DHCPv6 Settings:
  Prefix delegation: Enable
  DHCP User Class (Option 15): (leave blank unless ISP requires)
  DHCP Domain Name: (optional)

LAN:
  IPv6 Mode: Auto      — router assigns prefix automatically
  Enable RDNSS: Yes    — send DNS servers via RA
  IPv6 DNS Server 1: 2606:4700:4700::1111
  IPv6 DNS Server 2: 2001:4860:4860::8888
```

## Orbi Mesh IPv6

Orbi systems use the same Netgear web GUI via `orbilogin.com`.

```
Path: Advanced → Advanced Setup → IPv6

Orbi-specific notes:
  - Main Orbi router handles DHCPv6-PD from ISP
  - Satellite nodes receive IPv6 over backhaul
  - All nodes share single /64 prefix on LAN
  - IPv6 Firewall: enabled by default (block inbound)

LAN IPv6 Setup:
  Advertise IPv6 prefix to LAN: Enable
  Use DHCP for DNS: Enable (gets DNS from ISP DHCPv6)
```

## Verify Configuration via Router Diagnostics

Use the built-in diagnostics or telnet/SSH where available.

```bash
# Netgear routers: Advanced → Administration → Diagnostics

# Ping test to IPv6 address from router GUI:
# Target: 2606:4700:4700::1111
# Result should show 0% packet loss

# For routers with SSH (R7800, R9000 with custom firmware):
ssh admin@192.168.1.1

# Check WAN IPv6
ip -6 addr show dev eth0

# Check prefix delegation
ip -6 route show

# Check radvd is running
ps | grep radvd

# Check RA config
cat /tmp/radvd.conf | head -30
```

## Troubleshoot Common Issues

```bash
# Issue 1: "No IPv6 connection" in status page
# Cause: ISP not sending DHCPv6 offers, or wrong connection type
# Fix: Change from "Auto Detect" to explicit "DHCPv6" mode
# Then: Power cycle router (not just reboot)

# Issue 2: Router has IPv6 but LAN devices don't
# Check: Is "Enable IPv6 address on LAN" checked?
# Also verify: Firewall → IPv6 → not blocking RA (ICMPv6 type 134)

# Issue 3: IPv6 works but slow (MTU issue)
# Nighthawk with PPPoE: set MTU to 1452
# Path: Advanced → Setup → WAN Setup → MTU Size

# Issue 4: IPv6 drops after a few hours (lease renewal)
# Known issue on R7000 — update to latest firmware
# Or set shorter DHCP renewal interval

# Check status after changes
curl -6 https://ifconfig.co
```

## Testing IPv6 on LAN Devices

```bash
# From any device on the Netgear LAN

# Verify global IPv6 address
ip -6 addr show | grep "scope global"

# Verify default route
ip -6 route show default

# Test connectivity
ping6 -c 4 2606:4700:4700::1111

# Test DNS over IPv6
dig AAAA google.com @2606:4700:4700::1111 +short

# Verify internet-facing IPv6
curl -6 -s https://ifconfig.co
```

## Conclusion

Netgear Nighthawk and Orbi routers configure IPv6 under Advanced → Advanced Setup → IPv6. Select DHCPv6 as the connection type for most ISPs; Auto Detect also works for ISPs that use RA announcements. Enable prefix delegation and set up IPv6 LAN advertisement so that LAN devices receive /64 prefixes via SLAAC. Configure Cloudflare or Google IPv6 DNS servers for reliability. Keep the IPv6 firewall enabled — Netgear blocks all inbound by default, which is the correct stance for a home router. If the router acquires IPv6 but LAN devices do not, verify that RA advertisement to LAN is enabled and recheck firewall rules blocking ICMPv6.
