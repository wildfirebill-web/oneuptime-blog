# How to Configure IPv6 Router Advertisements on MikroTik

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MikroTik, RouterOS, Router Advertisement, SLAAC, Networking

Description: Configure IPv6 Router Advertisements on MikroTik RouterOS to enable SLAAC address autoconfiguration and DNS delivery for connected clients.

## Introduction

MikroTik RouterOS manages IPv6 Router Advertisements through the `/ipv6 nd` menu. The interface is straightforward but has some RouterOS-specific naming conventions to be aware of. This guide covers both CLI (using the RouterOS command line) and equivalent steps for Winbox.

## Prerequisites

- MikroTik router with RouterOS 6.x or 7.x
- IPv6 package enabled (`/system package` — verify `ipv6` is installed)
- IPv6 address assigned to the LAN interface

## Checking IPv6 Package Status

```bash
# Via RouterOS CLI
/system package print
# Verify "ipv6" package is installed and running

# Enable IPv6 forwarding
/ipv6 settings set forward=yes
```

## Assigning an IPv6 Address to the LAN Interface

```bash
# Assign a static IPv6 /64 to bridge1 (typical LAN interface)
/ipv6 address add address=2001:db8:1:1::1/64 interface=bridge1 advertise=yes

# Verify the address is assigned
/ipv6 address print
```

The `advertise=yes` flag tells RouterOS to include this prefix in Router Advertisements.

## Configuring Router Advertisement Settings

```bash
# Set RA parameters for the bridge1 interface
/ipv6 nd set [find interface=bridge1] \
    interface=bridge1 \
    ra-interval=30s-100s \
    ra-lifetime=30m \
    managed-address-configuration=no \
    other-configuration=no \
    advertise-dns=yes \
    advertise-mac-address=yes

# Verify the configuration
/ipv6 nd print detail
```

## Configuring RDNSS via RouterOS

MikroTik can advertise DNS servers in Router Advertisements:

```bash
# Add a DNS server to advertise in RA
/ipv6 nd prefix
# DNS is controlled separately via the dns-server setting

# RouterOS 7.x: Set the DNS to advertise via RA
/ipv6 nd set [find interface=bridge1] dns=2001:db8:1:1::53

# For multiple DNS servers, RouterOS uses the system DNS by default
# when advertise-dns=yes
/ip dns set servers=2001:db8:1:1::53,2606:4700:4700::1111
```

## Configuring RA Prefix Details

```bash
# Configure prefix options for RA (these are auto-derived from /ipv6 address)
# To customize:
/ipv6 nd prefix add prefix=2001:db8:1:1::/64 \
    interface=bridge1 \
    valid-lifetime=1d \
    preferred-lifetime=4h \
    autonomous=yes \
    on-link=yes
```

## Setting M/O Flags for DHCPv6

If you want clients to use DHCPv6 for addresses:

```bash
# M flag = 1: clients use DHCPv6 for addresses
/ipv6 nd set [find interface=bridge1] managed-address-configuration=yes

# O flag = 1: clients use DHCPv6 for DNS and other config only
/ipv6 nd set [find interface=bridge1] other-configuration=yes
```

## Disabling RA on WAN Interfaces

```bash
# Disable Router Advertisements on the ether1 (WAN) interface
/ipv6 nd set [find interface=ether1] disabled=yes
# Or add with disabled
/ipv6 nd add interface=ether1 disabled=yes
```

## Verifying Router Advertisements

```bash
# Show current ND/RA configuration
/ipv6 nd print detail

# Show neighbor table (clients that responded to RA)
/ipv6 neighbor print

# Monitor RA traffic with packet sniffer
/tool sniffer quick interface=bridge1 ip-protocol=icmpv6
```

## Example Full Configuration

```bash
# Complete MikroTik IPv6 RA setup from scratch

# 1. Assign IPv6 address
/ipv6 address add address=2001:db8:1:1::1/64 interface=bridge1 advertise=yes

# 2. Set system DNS
/ip dns set servers=2001:db8:1:1::53,2001:4860:4860::8888

# 3. Configure RA for the LAN
/ipv6 nd set [find interface=bridge1] \
    ra-interval=30s-100s \
    ra-lifetime=30m \
    managed-address-configuration=no \
    other-configuration=no \
    advertise-dns=yes

# 4. Verify
/ipv6 nd print detail
/ipv6 address print
```

## Conclusion

MikroTik RouterOS provides a concise CLI for configuring IPv6 Router Advertisements through the `/ipv6 nd` menu. Setting `advertise=yes` on the address and `advertise-dns=yes` on the ND profile covers most deployment needs. For DHCPv6 integrated deployments, set the M and O flags appropriately and configure a DHCPv6 server to complement the RA.
