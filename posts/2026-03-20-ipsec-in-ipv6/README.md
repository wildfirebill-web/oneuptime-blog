# How to Understand IPsec in IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, Security, AH, ESP

Description: Understand how IPsec works in the context of IPv6, including header placement, the change from mandatory to optional, and how AH and ESP extension headers integrate with the IPv6 packet structure.

## Overview

IPsec was originally mandatory for IPv6 (RFC 2460), but RFC 6434 (2011) changed this to optional since practical experience showed that making it mandatory created interoperability problems without meaningfully improving security. IPsec in IPv6 uses the same two protocols as IPv4 - Authentication Header (AH) and Encapsulating Security Payload (ESP) - but they are carried as IPv6 Extension Headers rather than as separate IP protocols.

## How IPsec Headers Appear in IPv6

In IPv6, AH and ESP are extension headers with Next Header values:

| Protocol | Next Header Value | Position in Header Chain |
|----------|------------------|--------------------------|
| AH | 51 | After Routing Header, before upper-layer |
| ESP | 50 | After Routing Header, before upper-layer |

```text
IPv6 Packet with AH (Transport Mode):
IPv6 Header (NH=51) → AH Header → TCP/UDP Payload

IPv6 Packet with ESP (Transport Mode):
IPv6 Header (NH=50) → ESP Header → [Encrypted TCP/UDP] → ESP Trailer → ESP Auth

IPv6 Packet with ESP (Tunnel Mode):
Outer IPv6 Header (NH=50) → ESP Header → [Inner IPv6 Header → TCP/UDP] → ESP Trailer → ESP Auth
```

## AH vs ESP in IPv6

| Feature | AH (NH=51) | ESP (NH=50) |
|---------|-----------|-------------|
| Authentication | Yes | Yes (optional) |
| Encryption | No | Yes |
| Protects IP header | Yes (most fields) | Only inner header (tunnel) |
| NAT compatible | No | Yes (in NAT-T mode) |
| Use case | Integrity only | Confidentiality + integrity |

## Transport Mode vs Tunnel Mode

### Transport Mode

Protects only the payload; original IP header is kept:

```text
Original: [IPv6: src=A, dst=B] [TCP: sport=1234, dport=443] [Data]

With ESP Transport Mode:
[IPv6: src=A, dst=B, NH=50] [ESP] [TCP: sport=1234, dport=443] [Data] [ESP-Trailer] [ESP-Auth]
```

Use case: Host-to-host communication on a trusted backbone.

### Tunnel Mode

Encapsulates the entire original packet:

```text
Original: [IPv6: src=A, dst=B] [TCP] [Data]

With ESP Tunnel Mode:
[Outer IPv6: src=GW1, dst=GW2, NH=50] [ESP] [Inner IPv6: src=A, dst=B] [TCP] [Data] [ESP-Trailer] [ESP-Auth]
```

Use case: Gateway-to-gateway VPN (site-to-site).

## The Mandatory-to-Optional Change (RFC 6434)

RFC 2460 (original IPv6) stated: "IPv6 requires support for AH and ESP"

RFC 6434 (2011) revised this to: "Implementations SHOULD support AH and ESP"

Reasons for the change:
- Many deployments never used IPsec
- TLS/DTLS/application-layer security covers most use cases
- Mandatory IPsec created pressure without benefit
- Interoperability with NAT required workarounds that undermined the "mandatory" nature

```bash
# Verify IPsec support on Linux

ip xfrm state   # Should work if IPsec is supported
lsmod | grep xfrm

# Verify kernel modules are loaded
modprobe xfrm4_mode_transport
modprobe xfrm6_mode_transport
modprobe xfrm6_mode_tunnel
```

## Security Associations (SA) for IPv6

IPsec SAs in IPv6 use the same concept as IPv4: identified by (SPI, Destination Address, Protocol):

```bash
# View current IPv6 IPsec SAs
ip xfrm state list | grep '::'   # IPv6 SAs have :: notation

# Sample SA output:
# src 2001:db8:1::1 dst 2001:db8:2::1
#     proto esp spi 0xc1234567 reqid 1 mode transport
#     auth hmac(sha256) 0xabc...
#     enc cbc(aes) 0xdef...
```

## IPv6 IPsec in Practice

Modern deployments typically use IKEv2 (RFC 7296) for SA negotiation:

```bash
# strongSwan: Basic IPv6 IKEv2 configuration
# /etc/swanctl/conf.d/ipv6-test.conf
connections {
    ipv6-host {
        version = 2
        local_addrs  = 2001:db8:1::1
        remote_addrs = 2001:db8:2::1

        local {
            auth = psk
            id = 2001:db8:1::1
        }
        remote {
            auth = psk
            id = 2001:db8:2::1
        }

        children {
            ipv6-host-to-host {
                local_ts  = 2001:db8:1::1/128
                remote_ts = 2001:db8:2::1/128
                mode = transport
                esp_proposals = aes256gcm128-prfsha256-ecp256
            }
        }
    }
}

secrets {
    ike-ipv6 {
        id-local  = 2001:db8:1::1
        id-remote = 2001:db8:2::1
        secret = "StrongPresharedKey123!"
    }
}
```

## IPv6 AH Complication: Mutable Header Fields

AH authenticates the entire IPv6 header, but some fields are mutable (can change in transit):

**Mutable fields (zeroed for AH calculation):**
- Traffic Class (DSCP may be changed by QoS devices)
- Flow Label (may be changed in transit)
- Hop Limit (decremented at each router)

**Immutable fields (authenticated by AH):**
- Version (always 6)
- Payload Length
- Source Address
- Destination Address (unless Routing Header present)

This means AH over IPv6 is still meaningful but doesn't protect mutable fields.

## Summary

IPsec in IPv6 uses AH (NH=51) and ESP (NH=50) as Extension Headers. IPsec was mandatory in original IPv6 (RFC 2460) but downgraded to SHOULD in RFC 6434 because application-layer security (TLS) serves most use cases. Transport mode protects host-to-host communications; tunnel mode is used for gateway VPNs. Modern deployments use IKEv2 (strongSwan/Libreswan) for automatic SA negotiation. AH authenticates the IPv6 header but must zero mutable fields (Traffic Class, Flow Label, Hop Limit) for the calculation.
