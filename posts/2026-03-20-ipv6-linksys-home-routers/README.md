# How to Configure IPv6 on Linksys Home Routers - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Linksys, Velop, Home Router, DHCPv6

Description: Configure IPv6 on Linksys WRT, EA, and Velop mesh routers including DHCPv6-PD setup, LAN SLAAC, and troubleshooting common connectivity problems.

## Supported Linksys Models

IPv6 is supported on Linksys WRT series (WRT3200ACM, WRT32X), EA series (EA6900, EA8300), and Velop mesh systems. Access via `myrouter.local` or `192.168.1.1`.

## GUI Configuration (EA/WRT Series)

```text
Path: Connectivity → Internet Settings → IPv6

IPv6 Mode options:
  Automatic Configuration - DHCPv6  - most common ISP type
  Automatic Configuration - SLAAC   - RA-only ISPs
  Static IPv6 Address               - fixed WAN address
  6to4                              - legacy tunnel (avoid)
  Disabled

For DHCPv6 (recommended for most ISPs):
  IPv6 Type: Automatic Configuration - DHCPv6
  DHCP-PD: Enabled
  Request Prefix Size: /56 or /48

LAN:
  Send Advertisement: Enable (enables SLAAC for LAN devices)
  DNS Mode: Automatic (from ISP) or Manual
  IPv6 DNS 1: 2606:4700:4700::1111
  IPv6 DNS 2: 2001:4860:4860::8888
```

## Velop Mesh IPv6

Linksys Velop uses the Linksys app and a simplified web interface.

```nginx
Linksys App → Wi-Fi → Advanced Settings → IPv6

Or web GUI at 192.168.1.1:
  Router Mode → Connectivity → IPv6

Velop Notes:
  - Parent node handles ISP DHCPv6-PD
  - Child nodes bridge IPv6 over backhaul
  - All nodes serve same /64 to wireless clients
  - IPv6 firewall: enabled by default

For bridge/AP mode (no routing):
  IPv6 passes through transparently from upstream router
```

## WRT Series with OpenWrt

WRT3200ACM and WRT32X support OpenWrt natively.

```bash
# Install OpenWrt on WRT series - check openwrt.org for image

# After installation, configure via LuCI or CLI

# Configure DHCPv6-PD client on WAN
uci set network.wan6.proto='dhcpv6'
uci set network.wan6.reqprefix='auto'
uci set network.wan6.reqaddress='try'
uci commit network
/etc/init.d/network restart

# Configure RA on LAN
uci set network.lan.ip6assign='64'
uci commit network

# Verify on OpenWrt
ip -6 addr show
ip -6 route show
```

## Troubleshoot Linksys IPv6

```bash
# Issue 1: IPv6 not detected - wrong mode selected
# Fix: Change to "Automatic Configuration - DHCPv6" and save

# Issue 2: Router shows IPv6 WAN but LAN devices have no IPv6
# Cause: RA advertisement disabled
# Fix: Enable "Send Advertisement" in LAN settings

# Issue 3: Velop child nodes not passing IPv6
# Cause: Backhaul not propagating RA
# Fix: Factory reset child nodes and re-pair

# Issue 4: IPv6 works on 5GHz but not 2.4GHz
# Usually MTU or RA interval issue
# Reduce RA interval to 30 seconds in Advanced settings

# Diagnostic from connected device
ping6 -c 3 2606:4700:4700::1111
traceroute6 2001:4860:4860::8888
curl -6 https://ifconfig.co
```

## Test from LAN Device

```bash
# Verify LAN device has received IPv6 from Linksys router

# Linux/macOS
ip -6 addr show | grep "scope global"
ip -6 route show default

# Windows PowerShell
Get-NetIPAddress -AddressFamily IPv6 | Where-Object PrefixOrigin -eq RouterAdvertisement

# Full connectivity test
ping6 2606:4700:4700::1111      # Cloudflare DNS
ping6 2001:4860:4860::8888      # Google DNS
curl -6 https://ipv6.google.com  # HTTP over IPv6
```

## Conclusion

Linksys routers configure IPv6 under Connectivity → Internet Settings → IPv6. Select "Automatic Configuration - DHCPv6" for most ISPs, enable DHCP-PD, and enable RA advertisement to the LAN. Velop mesh systems automatically propagate IPv6 from the parent node through backhaul to all child nodes. For advanced users, replace the firmware with OpenWrt on WRT-series routers for full `odhcp6c` prefix delegation and LuCI IPv6 management. If LAN devices do not receive IPv6 addresses, confirm that Router Advertisement is enabled in the LAN settings - Linksys sometimes requires this to be explicitly toggled on.
