# How to Configure IPv6 on MikroTik RouterOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MikroTik, RouterOS, Router, Firewall, Networking

Description: Configure IPv6 on MikroTik RouterOS including address assignment, DHCPv6 prefix delegation, firewall rules, and address lists for a complete IPv6 deployment.

## Introduction

MikroTik RouterOS supports full IPv6 with a comprehensive feature set including DHCPv6 client (prefix delegation), DHCPv6 server, Router Advertisements, and IPv6 firewall. This guide covers a complete IPv6 deployment on a MikroTik router.

## Step 1: Enable IPv6 Package

```bash
# Check that IPv6 package is installed
/system package print
# Look for "ipv6" in the list

# If not installed, download from MikroTik's package repository
# Packages > Download & Install > ipv6
```

## Step 2: Configure WAN IPv6 via DHCPv6-PD

```bash
# Configure DHCPv6 client on the WAN interface (ether1)
# Request a prefix delegation from the ISP
/ipv6 dhcp-client add \
    interface=ether1 \
    request=prefix \
    pool-name=ISP-POOL \
    pool-prefix-length=64 \
    add-default-route=yes \
    use-peer-dns=yes

# Verify DHCPv6 client status
/ipv6 dhcp-client print detail
```

## Step 3: Configure LAN IPv6 Address

```bash
# Assign IPv6 address to the LAN interface from the delegated pool
/ipv6 address add \
    interface=bridge1 \
    from-pool=ISP-POOL \
    advertise=yes

# Verify
/ipv6 address print
```

## Step 4: Configure Router Advertisements

```bash
# Set RA parameters for the LAN bridge
/ipv6 nd set [find interface=bridge1] \
    ra-interval=30s-100s \
    ra-lifetime=30m \
    managed-address-configuration=no \
    other-configuration=no \
    advertise-dns=yes \
    advertise-mac-address=yes

# Set system DNS for advertisement
/ip dns set servers=2606:4700:4700::1111,2001:4860:4860::8888
```

## Step 5: Configure IPv6 Firewall

MikroTik's IPv6 firewall uses the same structure as IPv4 with the `/ipv6 firewall` menu:

```bash
# Input chain - protect the router itself
/ipv6 firewall filter add chain=input action=accept connection-state=established,related comment="Accept established/related"
/ipv6 firewall filter add chain=input action=accept protocol=icmpv6 comment="Accept ICMPv6 (required)"
/ipv6 firewall filter add chain=input action=accept src-address=fe80::/10 comment="Accept link-local"
/ipv6 firewall filter add chain=input action=drop comment="Drop all other input"

# Forward chain - traffic through the router
/ipv6 firewall filter add chain=forward action=accept connection-state=established,related comment="Accept established"
/ipv6 firewall filter add chain=forward action=accept protocol=icmpv6 comment="Accept ICMPv6"
/ipv6 firewall filter add chain=forward action=accept in-interface=bridge1 comment="Allow LAN to WAN"
/ipv6 firewall filter add chain=forward action=drop in-interface=ether1 comment="Drop unsolicited inbound from WAN"
```

## Step 6: Configure IPv6 Address Lists

For complex firewall rules, use address lists:

```bash
# Create an address list for internal IPv6 prefixes
/ipv6 firewall address-list add address=2001:db8::/48 list=INTERNAL comment="Internal prefix"
/ipv6 firewall address-list add address=fc00::/7 list=INTERNAL comment="ULA range"
/ipv6 firewall address-list add address=fe80::/10 list=LINK-LOCAL comment="Link-local"

# Use address list in firewall rule
/ipv6 firewall filter add \
    chain=forward \
    dst-address-list=INTERNAL \
    in-interface=ether1 \
    action=drop \
    comment="Block WAN traffic to internal addresses"
```

## Step 7: Configure Static IPv6 Routes

```bash
# Add a static IPv6 route
/ipv6 route add dst-address=2001:db8:remote::/48 gateway=2001:db8:0:1::2

# Add a floating default route for failover (higher distance)
/ipv6 route add dst-address=::/0 gateway=2001:db8:backup::1 distance=200
```

## Verification

```bash
# Show all IPv6 addresses
/ipv6 address print

# Show IPv6 routing table
/ipv6 route print

# Show DHCPv6 client status
/ipv6 dhcp-client print

# Show neighbor discovery cache
/ipv6 neighbor print

# Show firewall rules with hit counters
/ipv6 firewall filter print stats

# Test connectivity
/ping 2606:4700:4700::1111 count=3
```

## Conclusion

MikroTik RouterOS provides a comprehensive IPv6 stack that is well-suited for both home and enterprise deployments. DHCPv6-PD client simplifies ISP prefix acquisition, the RA configuration integrates tightly with the address pool, and the IPv6 firewall mirrors the familiar IPv4 firewall structure. Use address lists to keep firewall rules DRY (Don't Repeat Yourself) when working with multiple IPv6 prefixes or address ranges.
