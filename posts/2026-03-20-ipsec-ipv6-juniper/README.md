# How to Configure IPsec IPv6 on Juniper Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, Juniper, Junos, VPN

Description: Learn how to configure IPv6 IPsec site-to-site VPNs on Juniper routers using JunOS, including IKEv2 policy, security associations, and route-based VPN configuration.

## Overview

Juniper JunOS supports IPv6 IPsec using IKEv2 with route-based VPN configuration. Route-based VPNs use a secure tunnel interface (st0) that behaves like a regular interface, making routing straightforward. JunOS IPv6 IPsec configuration follows the same pattern as IPv4 but with IPv6 gateway addresses.

## Full IPv6 IPsec Configuration

### IKEv2 Proposal and Policy

```text
# IKEv2 Proposal (cipher suites)

set security ike proposal IKEv2-PROP
    authentication-method pre-shared-keys
    dh-group group14
    authentication-algorithm sha-256
    encryption-algorithm aes-256-cbc
    lifetime-seconds 28800

# IKEv2 Policy
set security ike policy IKEv2-POLICY
    mode main
    proposals IKEv2-PROP
    pre-shared-key ascii-text "StrongSharedKey123!"

# IKEv2 Gateway (remote peer)
set security ike gateway IPV6-GATEWAY
    ike-policy IKEv2-POLICY
    address 2001:db8:gw2::1
    local-address 2001:db8:gw1::1
    version v2-only
    local-identity inet6 2001:db8:gw1::1
    remote-identity inet6 2001:db8:gw2::1
```

### IPsec Proposal and Policy

```text
# IPsec Proposal (ESP transform)
set security ipsec proposal ESP-PROP
    protocol esp
    authentication-algorithm hmac-sha-256-128
    encryption-algorithm aes-256-cbc
    lifetime-seconds 3600

# IPsec Policy
set security ipsec policy IPSEC-POLICY
    proposals ESP-PROP

# IPsec VPN
set security ipsec vpn IPV6-VPN
    bind-interface st0.0
    ike {
        gateway IPV6-GATEWAY
        ipsec-policy IPSEC-POLICY
    }
    establish-tunnels immediately
```

### Secure Tunnel Interface (st0)

```text
# Create and configure the tunnel interface
set interfaces st0 unit 0 family inet6 address 2001:db8:vti::1/64
set interfaces st0 unit 0 description "IPv6 VPN to Site2"

# Route Site2 traffic through the tunnel
set routing-options rib inet6.0 static route 2001:db8:site2::/48 next-hop st0.0
```

### Security Zones and Policies

```text
# Place tunnel interface in VPN zone
set security zones security-zone VPN interfaces st0.0

# Allow traffic from local network to VPN zone
set security policies from-zone INTERNAL to-zone VPN policy ALLOW-TO-SITE2
    match source-address site1-network
    match destination-address site2-network
    match application any
set security policies from-zone INTERNAL to-zone VPN policy ALLOW-TO-SITE2
    then permit

# Reverse direction
set security policies from-zone VPN to-zone INTERNAL policy ALLOW-FROM-SITE2
    match source-address site2-network
    match destination-address site1-network
    match application any
set security policies from-zone VPN to-zone INTERNAL policy ALLOW-FROM-SITE2
    then permit

# Address book entries
set security address-book global address site1-network 2001:db8:site1::/48
set security address-book global address site2-network 2001:db8:site2::/48
```

## AES-GCM (AEAD) Configuration

```text
# Use AES-GCM for better performance (single-pass auth+encryption)
set security ipsec proposal ESP-GCM-PROP
    protocol esp
    encryption-algorithm aes-256-gcm
    lifetime-seconds 3600

# Note: With GCM, no separate authentication-algorithm needed
```

## Verification Commands

```text
# Show IKEv2 security associations
show security ike security-associations

# Sample output:
# Index   Remote Address            State      Initiator cookie  Responder cookie  Mode
# 1       2001:db8:gw2::1           UP         a1b2c3d4e5f60718  8877665544332211  IKEv2

# Show IPsec security associations
show security ipsec security-associations

# Sample output:
# Total active tunnels: 1
# ID    Algorithm        SPI      Life:sec/kb  Mon         vsys
# <1>   ESP:aes256/sha256 abc12345 3558/ unlim  -           root

# Show VPN statistics
show security ipsec statistics

# Show IPsec SA detail
show security ipsec security-associations detail

# Test connectivity
ping6 2001:db8:site2::1 routing-instance default count 5
```

## Troubleshooting

```text
# Clear and re-establish
clear security ike security-associations
clear security ipsec security-associations

# Debug IKEv2 negotiation
set security ike traceoptions file ike-debug.log
set security ike traceoptions flag all

# View debug log
monitor start ike-debug.log

# Common issues:
# "IKE SA not found" → policy or address mismatch
# "Proposal not accepted" → cipher mismatch between peers
# Check with: show security ike statistics
```

## Summary

Juniper IPv6 IPsec uses a three-tier configuration: IKEv2 proposal/policy/gateway for IKE, IPsec proposal/policy/VPN for ESP, and a secure tunnel interface (st0) for routing. Route-based VPN binds the IPsec VPN to st0, and routing policies direct site-to-site traffic through it. Security zone policies control which traffic flows between zones. Use `show security ike security-associations` and `show security ipsec security-associations` to verify tunnel status. Use `establish-tunnels immediately` to initiate the tunnel on commit rather than waiting for traffic.
