# How to Configure IPv6 on TP-Link Home Routers - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, TP-Link, Home Router, DHCPv6, SLAAC

Description: Enable IPv6 on TP-Link Archer and Deco series routers, configure DHCPv6-PD, SLAAC for LAN devices, and troubleshoot common IPv6 connectivity issues.

## Supported TP-Link Models

Most TP-Link Archer series (C6, AX20, AX50, AX73, AX6000) and Deco mesh systems support IPv6. Check: Advanced → IPv6 in the Tether app or web GUI.

## GUI Configuration (Archer Series)

Access the router web interface at `192.168.0.1` or `tplinkwifi.net`.

```text
Path: Advanced → IPv6

WAN IPv6 Connection Type: DHCPv6
  (Most ISPs - automatically requests address and prefix)

Alternatively for SLAAC ISPs:
  WAN Type: SLAAC
  DNS Server: Get IPv6 DNS from ISP (enabled)
  or Manual DNS:
    Primary: 2606:4700:4700::1111
    Secondary: 2001:4860:4860::8888

LAN settings:
  Assign IPv6 address: Enable
  IPv6 Prefix: (auto-filled from delegated prefix)
  IPv6 Prefix Length: 64
  DHCPv6 Server: Enable (stateless - SLAAC + DNS via RA)

Enable IPv6 Firewall: Yes (recommended)
```

## TP-Link Deco (Mesh) IPv6

Deco systems configure IPv6 through the Tether mobile app.

```sql
Tether App → Select Deco network → More → Advanced → IPv6

IPv6 Status: Enable

Internet Connection:
  Type: DHCPv6 (recommended for most ISPs)

LAN:
  IPv6 Address Assignment: Enabled
  DNS: Automatic (from ISP) or Manual

Note: Deco mesh nodes relay IPv6 from the main node.
All nodes share the same /64 prefix for the LAN.
```

## Verify via Router Admin CLI

TP-Link OpenWrt-based routers support SSH for advanced verification.

```bash
# Enable SSH: Advanced → System → Remote Management → SSH

# or via Telnet on older models

# Connect to router
ssh root@192.168.0.1

# Check WAN IPv6
ip -6 addr show dev pppoe-wan 2>/dev/null || ip -6 addr show dev eth0.2

# Check delegated prefix routing
ip -6 route show

# Check LAN RA is running
ps | grep radvd

# View RA config
cat /tmp/radvd.conf
```

## Troubleshooting TP-Link IPv6

Common issues and quick fixes.

```bash
# Issue 1: Router shows IPv6 address but devices don't get one
# Fix: Ensure RA is enabled and LAN assignment is on
# Check radvd is running:
ps | grep radvd
# Restart if missing:
/etc/init.d/radvd restart

# Issue 2: IPv6 intermittently drops
# Often DHCPv6 lease renewal failure
# Force DHCPv6 renew from WAN interface:
ip -6 addr flush dev eth0.2
udhcpc6 -i eth0.2 -P 60 &

# Issue 3: Some sites unreachable over IPv6 (MTU issue)
# TP-Link PPPoE MTU default is 1480
# Check:
ip link show dev pppoe-wan | grep mtu
# Reduce if needed:
ip link set dev pppoe-wan mtu 1452

# Issue 4: IPv6 firewall blocking too aggressively
# Review firewall rules:
ip6tables -L -n -v
# Temporarily disable to test:
ip6tables -P FORWARD ACCEPT
```

## Test IPv6 From LAN Device

After configuring the router, verify from a device on the network.

```bash
# From a PC connected to TP-Link router

# Check device has IPv6
ip -6 addr show | grep "scope global"

# Ping router's LAN IPv6
ping6 2001:db8:home:1::1    # substitute actual router LAN IPv6

# Ping internet
ping6 2606:4700:4700::1111

# Check DNS works over IPv6
dig AAAA example.com @2606:4700:4700::1111

# Verify public IPv6 address
curl -6 https://ifconfig.co
```

## Conclusion

TP-Link Archer and Deco series routers configure IPv6 under Advanced → IPv6 in the web GUI or Tether app. Select DHCPv6 for the WAN connection type; the router will negotiate a prefix delegation from the ISP and automatically configure radvd to distribute /64 prefixes to LAN devices. Set the IPv6 firewall to enabled to block unsolicited inbound connections. If devices on the LAN do not receive IPv6 addresses, confirm that DHCPv6 LAN assignment is enabled and that radvd is running on the router. MTU issues are common on PPPoE - set the WAN MTU to 1452 if large IPv6 packets fail.
