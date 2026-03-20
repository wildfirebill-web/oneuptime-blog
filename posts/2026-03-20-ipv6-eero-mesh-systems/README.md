# How to Configure IPv6 on eero Mesh Systems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, eero, Mesh Network, Amazon, DHCPv6

Description: Enable IPv6 on Amazon eero mesh systems, verify DHCPv6-PD prefix delegation, and troubleshoot IPv6 connectivity for home networks using eero as the router.

## eero IPv6 Overview

eero (owned by Amazon) supports native IPv6 with automatic DHCPv6-PD configuration. All settings are managed through the eero mobile app — there is no traditional web GUI.

## Enable IPv6 in eero App

```
eero App → Settings → Advanced Settings → IPv6 → Enable

eero IPv6 behavior:
  - Requests WAN IPv6 address via DHCPv6
  - Requests prefix delegation (/56 recommended, accepts /60)
  - Distributes /64 via SLAAC to all clients
  - All eero nodes in the mesh share a single /64 prefix
  - Uses SLAAC (no DHCPv6 for clients)

Note: eero does NOT support:
  - Static WAN IPv6 address
  - 6in4 or 6to4 tunnels
  - Manual prefix configuration
  - DHCPv6 for LAN clients (SLAAC only)
```

## Verify IPv6 from a Connected Device

Since eero has no CLI, test from a device on the network.

```bash
# From any device on the eero network

# Check for global IPv6 address assigned via SLAAC
ip -6 addr show | grep "scope global"
# Should show: inet6 2001:db8:XXXX:XXXX:YYYY:YYYY:YYYY:YYYY/64 scope global dynamic

# Check for IPv6 default route
ip -6 route show default

# Test internet reachability
ping6 -c 4 2606:4700:4700::1111    # Cloudflare
ping6 -c 4 2001:4860:4860::8888    # Google

# Check public IPv6 address
curl -6 https://ifconfig.co

# Test IPv6 DNS resolution
dig AAAA google.com @2606:4700:4700::1111
```

## eero with ISP Modem in Bridge Mode

For eero to receive DHCPv6-PD, the ISP modem must be in bridge mode.

```
Step 1: Set ISP modem to bridge/passthrough mode
  - Log into modem admin page
  - Find "Bridge Mode" or "IP Passthrough" or "DMZ"
  - Set eero MAC address as passthrough target
  - Save and reboot modem

Step 2: eero automatically requests IPv6 from ISP
  - DHCPv6-PD negotiation happens automatically
  - Prefix appears in eero app: Advanced → IPv6 Status

Step 3: Verify in eero app
  eero App → Settings → Network Details
  Shows: IPv6 WAN Address and LAN prefix
```

## Amazon eero Pro 6 / 6E IPv6 Details

```bash
# eero Pro 6/6E supports Thread for Matter smart home devices
# Thread uses IPv6 (ULA addresses internally)
# eero acts as Thread border router

# From a Thread device (e.g., smart bulb, sensor):
# Device gets ULA IPv6 for mesh communication
# Border router maps to global IPv6 for internet access

# Check eero Thread status:
eero App → Devices → [smart home device] → Details
# Shows Thread IPv6 address

# Regular Wi-Fi clients get global IPv6 via SLAAC:
# Different /64 prefix from ISP delegation

# To check all IPv6 devices on eero network:
eero App → Devices → filter by "Wi-Fi" or "Wired"
# Tap any device to see its IPv6 address
```

## Troubleshoot eero IPv6

```bash
# Issue 1: IPv6 disabled or not working after enabling
# Fix: Power cycle eero gateway (unplug for 30 seconds)
# Then check: Settings → Advanced → IPv6 → Status

# Issue 2: ISP modem blocking DHCPv6-PD
# eero gateway shows no IPv6 WAN address
# Fix: Enable bridge mode on modem, or contact ISP

# Issue 3: eero has IPv6 WAN but clients not getting addresses
# Cause: eero RA not reaching all mesh nodes
# Fix: Reboot all eero nodes (app → restart all nodes)

# Issue 4: IPv6 works but some sites unreachable
# Common with MTU issues on PPPoE ISPs
# eero sets MTU automatically — limited control
# Workaround: use IPv6 PMTUD-aware DNS (Cloudflare 2606:4700:4700::1111)

# Verify fix from connected device
ping6 -c 3 2606:4700:4700::1111
curl -6 https://ipv6.google.com
```

## eero in Bridge Mode

If eero is behind another router, use eero in bridge mode.

```
eero App → Settings → Advanced Settings → eero Mode → Bridge Mode

In bridge mode:
  - eero acts as a managed AP only
  - Your upstream router handles DHCPv6-PD
  - IPv6 RA from upstream router passes through eero
  - eero Thread border router still works for Matter devices
  - eero Security+ features still apply to clients
```

## Conclusion

eero mesh systems enable IPv6 automatically once enabled in the eero app under Advanced Settings → IPv6. The gateway eero requests DHCPv6-PD from the ISP modem (which must be in bridge mode) and distributes a /64 prefix to all connected devices via SLAAC. eero Pro 6/6E also supports Thread, providing IPv6 connectivity for Matter smart home devices through a built-in Thread border router. Since eero has no traditional CLI, verify IPv6 from connected devices using `ping6` and `curl -6`. If clients are not receiving IPv6, reboot all eero nodes and verify the ISP modem is in bridge mode to allow DHCPv6-PD to reach the eero gateway.
