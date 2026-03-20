# How to Configure IPv6 on TP-Link Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, TP-Link, Router, DHCPv6, SLAAC, Home Network

Description: Configure IPv6 on TP-Link routers through the web interface, enabling DHCPv6 prefix delegation from your ISP and SLAAC for home network devices.

## Introduction

TP-Link routers (including Archer and Deco series) support IPv6 through both their web interface and the Tether mobile app. This guide covers enabling IPv6 WAN connectivity via DHCPv6-PD and configuring the LAN for SLAAC-based client addressing.

## Prerequisites

- TP-Link router with firmware supporting IPv6 (check router model's specifications)
- ISP that provides IPv6 connectivity
- Access to the router's admin interface at `http://192.168.0.1` or `http://tplinkwifi.net`

## Step 1: Configure WAN IPv6

1. Log in to the router admin interface
2. Navigate to **Advanced > IPv6**
3. Toggle **IPv6** to **On** (if shown)
4. Under **Internet Connection Type**, select:
   - **Dynamic IP (SLAAC/DHCPv6)**: For ISPs that auto-provision IPv6
   - **PPPoEv6**: For DSL/fiber ISPs using PPPoE
   - **Static IPv6**: If your ISP provided a static IPv6 address
5. For **Dynamic IP**:
   - Enable **DHCP-PD** (Prefix Delegation): Typically enabled by default
   - Set **Prefix Length**: `/56` or `/48` depending on your ISP
6. Click **Save**

## Step 2: Configure LAN IPv6

1. On the same **IPv6** settings page, scroll to **IPv6 LAN Settings**
2. Set **Mode** to:
   - **SLAAC**: Clients autoconfigure addresses (recommended for home use)
   - **SLAAC + Stateless DHCPv6**: Clients use SLAAC for addresses but DHCPv6 for DNS
   - **Stateful DHCPv6**: Clients use DHCPv6 for all configuration
3. For SLAAC mode:
   - **Prefix Length**: `/64` (auto-derived from delegated prefix)
   - **Site Prefix Type**: Use Delegated Prefix (from WAN DHCPv6-PD)
4. Set **DNS** servers if not using ISP-provided DNS:
   - Primary: `2606:4700:4700::1111`
   - Secondary: `2001:4860:4860::8888`
5. Click **Save**

## Step 3: Verify on the Router Status Page

1. Navigate to **Advanced > System Tools > Diagnostics** or **Status > IPv6**
2. Check that the WAN IPv6 address shows a valid global unicast address (not just link-local)
3. Verify the delegated prefix appears in the LAN section

## Step 4: Verify on Client Devices

From a Windows client:
```cmd
ipconfig /all
:: Look for IPv6 addresses under the network adapter
:: A valid global unicast address (2001:... or similar) should appear
```

From a Linux client:
```bash
# Check for global IPv6 address

ip -6 addr show scope global

# Test connectivity
ping6 2606:4700:4700::1111

# Verify DNS works over IPv6
nslookup -type=AAAA example.com
```

## Common TP-Link IPv6 Issues

**Issue: IPv6 option not visible in settings**

Some TP-Link firmware versions hide IPv6 under different menus:
- Try **Advanced > IPv6** or **Advanced > Network > IPv6**
- If still not visible, update the router firmware
- Check the TP-Link product page to confirm your model supports IPv6

**Issue: WAN gets a link-local address only (no global)**
- The ISP may not be providing IPv6 on your plan
- Check with your ISP if IPv6 is available and if it needs to be enabled on your account
- Some ISPs require native IPv6 rather than DHCPv6-PD

**Issue: LAN devices don't get IPv6 addresses**
```bash
# From a client, manually request an RA
rdisc6 eth0

# Check if M or O flags are set (indicates DHCPv6 needed)
# If Stateful address conf = Yes but no DHCPv6 server, addresses won't be assigned
```

## TP-Link Deco (Mesh) Notes

For TP-Link Deco mesh systems:
1. Open the **Tether** app
2. Go to **More > Advanced > IPv6**
3. Enable IPv6 and select the connection type
4. The mesh nodes automatically propagate IPv6 settings

## Conclusion

TP-Link routers provide a straightforward web-based IPv6 configuration experience. The key settings are the WAN connection type (typically Dynamic IP/DHCPv6-PD) and the LAN mode (SLAAC for simplicity, or stateful DHCPv6 for address control). Once configured, all connected devices receive globally routable IPv6 addresses that enable direct IPv6 communication without NAT.
