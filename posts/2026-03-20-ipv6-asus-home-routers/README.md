# How to Configure IPv6 on ASUS Home Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ASUS, Home Router, DHCPv6, Router Configuration

Description: Enable and configure IPv6 on ASUS routers running ASUSWRT firmware, including DHCPv6-PD, SLAAC, DNS settings, and firewall configuration.

## ASUS Router IPv6 Overview

ASUS routers running ASUSWRT (and Merlin) support multiple IPv6 connection types. Most ISPs use Native (DHCPv6) or DHCPv6 with prefix delegation.

## GUI Configuration

Navigate to the IPv6 settings page in the ASUS web interface.

```
Path: Advanced Settings → IPv6

Connection Type options:
  - Native               — DHCPv6 WAN + SLAAC on LAN (most ISPs)
  - Native with DHCP-PD  — Explicit prefix delegation (/56 or /48)
  - Static IPv6          — Fixed WAN IPv6 address (business)
  - 6in4 Static          — Hurricane Electric tunnel
  - Automatic 6to4       — Legacy tunnel (avoid)
  - FLET's IPv6 Service  — Japan NTT specific

Recommended settings for most ISPs:
  Connection Type: Native with DHCP-PD
  DHCP-PD: Enabled
  DNS Server 1: 2606:4700:4700::1111   (Cloudflare)
  DNS Server 2: 2001:4860:4860::8888   (Google)
  Enable Router Advertisement: Yes
  Enable DHCPv6 Server: Yes (stateless)
  LAN IPv6 prefix: (auto-filled from PD)
```

## ASUSWRT-Merlin CLI Configuration

Advanced configuration via SSH on Merlin firmware.

```bash
# SSH into ASUS router (enable SSH in Administration → System)
ssh admin@192.168.1.1

# Check current IPv6 WAN address
ip -6 addr show dev eth0   # WAN interface

# Check delegated prefix
ip -6 route show | grep "pref"

# Check LAN prefix
ip -6 addr show dev br0    # LAN bridge

# View radvd configuration (auto-generated)
cat /etc/radvd.conf

# Check DHCPv6 client log
cat /tmp/odhcp6c.log 2>/dev/null | tail -20
```

## Custom radvd Configuration (Merlin)

For advanced users who need custom RA settings.

```bash
# /jffs/configs/radvd.conf.add — extra radvd options appended by Merlin
# (do not replace the whole file)

interface br0 {
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;
    AdvLinkMTU 1480;    # For PPPoE connections

    RDNSS 2606:4700:4700::1111 2001:4860:4860::8888 {
        AdvRDNSSLifetime 300;
    };
};
```

## IPv6 Firewall on ASUS

ASUS routers have an IPv6 firewall that may block inbound connections.

```
GUI Path: IPv6 → IPv6 Firewall

Default: Block all inbound IPv6 connections
Options:
  - Disable (allow all inbound — not recommended)
  - Enable (block all inbound)
  - Custom rules via IPv6 Firewall tab

To allow a specific service (e.g., SSH on a home server):
  Service Name: SSH_inbound
  Protocol: TCP
  Source IP: ::/0 (any)
  Dest IP: 2001:db8:home:1::server
  Port: 22
```

## Testing IPv6 on ASUS Router

Verify everything is working from the router's console.

```bash
# SSH into router and run tests

# Check WAN IPv6 address
ip -6 addr show dev eth0 | grep "scope global"

# Ping upstream gateway
ping6 -c 3 $(ip -6 route show default | awk '{print $3}')

# Check internet reachability
ping6 -c 3 2606:4700:4700::1111

# Check DNS
nslookup -type=AAAA ipv6.google.com 2606:4700:4700::1111

# Count LAN devices with IPv6
ip -6 neigh show dev br0 | grep -v "fe80" | wc -l
echo "LAN devices with global IPv6"
```

## Conclusion

ASUS routers using ASUSWRT or Merlin firmware configure IPv6 via the IPv6 section of the web GUI. Select "Native with DHCP-PD" for most ISPs to enable DHCPv6-PD prefix delegation. The router automatically runs radvd to distribute the delegated /64 prefix to LAN devices via SLAAC. Set Cloudflare (2606:4700:4700::1111) or Google (2001:4860:4860::8888) as IPv6 DNS servers. Keep the IPv6 firewall enabled and add explicit inbound rules only for services you intentionally expose. For advanced customization, use Merlin firmware with `/jffs/configs/radvd.conf.add` and custom scripts.
