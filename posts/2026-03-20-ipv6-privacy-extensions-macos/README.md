# How to Configure IPv6 Privacy Extensions on macOS - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Privacy Extensions, macOS, Security, Networking, Terminal

Description: A guide to checking and configuring IPv6 privacy extensions on macOS, including verifying temporary address generation, system preferences configuration, and command-line management.

macOS has supported IPv6 privacy extensions since OS X 10.7 Lion. By default, macOS generates temporary IPv6 addresses for outbound connections, protecting privacy by not exposing the MAC address in the IPv6 address. This guide explains how to verify and manage this behavior.

## Checking Current IPv6 Addresses on macOS

```bash
# List all IPv6 addresses on active interfaces

ifconfig | grep "inet6"

# More detailed view with interface names
ifconfig en0 | grep "inet6"

# Expected output for a properly configured system:
# inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
# inet6 2001:db8:x:x:a1b2:c3d4:e5f6:7890 prefixlen 64 autoconf temporary
# inet6 2001:db8:x:x:aabb:ccff:fedd:eeff prefixlen 64 autoconf secured

# Look for "temporary" - this is the privacy extension address used for outbound
# Look for "secured" - this is the stable privacy address (RFC 7217)
```

## macOS IPv6 Address Types

macOS generates multiple types of IPv6 addresses:

```bash
# "temporary" = privacy extension address (RFC 8981)
#   Random interface ID, rotates periodically
#   Used for outbound connections by default

# "secured" = stable privacy address (RFC 7217)
#   Stable per interface and network, but doesn't expose MAC address
#   Used for inbound connections (servers reaching back to you)

# "autoconf" flag means the address was assigned via SLAAC

# Check via networksetup
networksetup -getinfo "Wi-Fi"
networksetup -getinfo "Ethernet"
```

## Configuring IPv6 via System Preferences / Settings

```bash
# macOS 13 (Ventura) and later: System Settings > Network > [Interface] > Details
# IPv6 Configure: Automatically (uses SLAAC with privacy extensions)

# macOS 12 (Monterey) and earlier: System Preferences > Network > Advanced > TCP/IP
# IPv6 Configure: Automatically

# When set to "Automatically", macOS:
# 1. Receives Router Advertisement with prefix
# 2. Generates a "secured" stable address using RFC 7217
# 3. Generates a "temporary" random address for outbound connections
```

## Managing IPv6 Privacy via Command Line

```bash
# Check IPv6 configuration for Wi-Fi interface
networksetup -getv6transporttype "Wi-Fi"

# Set IPv6 to automatic (enables SLAAC with privacy extensions)
networksetup -setv6automatic "Wi-Fi"

# Check current IPv6 addressing
scutil --nwi | grep "IPv6"

# View routing table (shows which source IP is preferred for outbound)
netstat -rn -f inet6 | head -20
```

## Verifying Privacy Extensions Are Working

```bash
# Check which address is used for outbound connections to the internet
curl -6 https://ipv6.icanhazip.com

# This should return your "temporary" address, not your MAC-derived EUI-64 address

# Verify with detailed curl output
curl -6 -v https://ipv6.icanhazip.com 2>&1 | grep "Connected to"

# Check if your address contains "ff:fe" (indicates EUI-64, NOT private)
ifconfig en0 | grep "inet6.*temporary"
# If this shows an address with ff:fe in the middle, privacy extensions may not be working

# List all your current global IPv6 addresses
ifconfig en0 | grep "inet6.*autoconf\|inet6.*temporary\|inet6.*secured"
```

## Address Rotation on macOS

```bash
# macOS rotates temporary addresses by default
# Check the current preferred lifetime
# (macOS doesn't expose this directly, but it's typically 24 hours)

# Force a new temporary address (disconnect and reconnect from Wi-Fi)
# Or via networksetup:
networksetup -setv6off "Wi-Fi"
networksetup -setv6automatic "Wi-Fi"

# After reconnecting, check new addresses
ifconfig en0 | grep "inet6.*temporary"

# You should see a different temporary address than before
```

## macOS Privacy Extensions vs VPN

```bash
# When connected to a VPN (OpenVPN, WireGuard, etc.):
# Check if the VPN interface also gets IPv6
ifconfig utun0 | grep "inet6"   # OpenVPN
ifconfig utun1 | grep "inet6"   # WireGuard (varies)

# Verify outbound traffic uses VPN's IPv6
curl -6 https://ipv6.icanhazip.com
# Should show the VPN provider's IPv6, not your ISP's

# Check if there's an IPv6 leak (macOS may use the non-VPN IPv6)
# If the result shows your ISP's prefix, you have an IPv6 leak
```

## Disabling IPv6 Privacy Extensions (Not Recommended for Clients)

```bash
# Disable IPv6 entirely for an interface (not recommended)
networksetup -setv6off "Wi-Fi"

# To use a static IPv6 address instead
networksetup -setv6manual "Wi-Fi" "2001:db8::mac1" 64 "2001:db8::1"

# Re-enable automatic (with privacy extensions)
networksetup -setv6automatic "Wi-Fi"
```

## Checking macOS Privacy in macOS Sequoia (15.x)

```bash
# Privacy Report in macOS tracks web trackers, not IPv6
# For IPv6 tracking prevention, check via Terminal:
ifconfig en0 | grep -E "temporary|secured"

# The presence of "temporary" address and absence of EUI-64
# (addresses without ff:fe in interface ID) confirms privacy extensions work

# Check with a test:
# 1. Disconnect from Wi-Fi and reconnect
# 2. Note the temporary address: ifconfig en0 | grep temporary
# 3. Disconnect and reconnect again
# 4. Check if temporary address changed: ifconfig en0 | grep temporary
# Addresses should be different (though rotation isn't immediate in all cases)
```

macOS implements IPv6 privacy extensions by default using a combination of RFC 7217 stable privacy addresses (the "secured" addresses) and RFC 8981 temporary addresses. Outbound connections use the temporary address, protecting against cross-network tracking. No additional configuration is needed on typical macOS client systems.
