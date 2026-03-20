# How to Understand IPv6 Extension Header-Based Evasion Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Security, Extension Headers, Evasion, Firewall

Description: Understand how attackers use IPv6 extension headers to evade security devices, and learn defensive measures to detect and block extension header-based attacks.

## Overview

IPv6 extension headers are a fundamental protocol feature that can also be abused to evade firewalls, IDS, and deep packet inspection tools. RFC 7045 and RFC 7112 document these concerns and define expected router and firewall behavior.

## IPv6 Extension Header Chain

IPv6 uses a chain of headers, each pointing to the next via the "Next Header" field:

```text
IPv6 Header → Hop-by-Hop Options → Destination Options → Routing Header
           → Fragment Header → Authentication Header → ESP → Payload
```

Each header has a fixed or variable length, making offset computation complex for security devices.

## How Extension Headers Enable Evasion

### 1. Non-Initial Fragment Attack

A firewall may only inspect the first fragment of a packet (which contains the transport layer header). Subsequent fragments contain payload only - no port numbers. Attackers can send a very large Routing Header in the first fragment to push the TCP/UDP header beyond what the firewall inspects.

### 2. Hop-by-Hop Options Header DoS

The Hop-by-Hop Options header is processed by every router on the path:

```text
An attacker sends packets with crafted Hop-by-Hop Options headers
→ Every router must process them → CPU exhaustion
```

```bash
# Detect and filter Hop-by-Hop headers

ip6tables -A INPUT -m ipv6header --header hop-by-hop -j DROP
# This drops all packets with Hop-by-Hop headers (RFC 6192 recommendation for hosts)
```

### 3. Routing Header Type 0 (RH0) - Deprecated

Type 0 Routing Headers allowed attackers to amplify traffic (packet magnification):

```bash
# Block deprecated Type 0 Routing Headers
ip6tables -A INPUT  -m rt --rt-type 0 -j DROP
ip6tables -A FORWARD -m rt --rt-type 0 -j DROP
```

RH0 is deprecated by RFC 5095 and should be blocked on all equipment.

### 4. Unknown Extension Header Evasion

Some security devices skip unknown Next Header values, allowing the payload to bypass inspection:

```bash
# nftables: Drop packets with extension headers that aren't expected
nft add rule ip6 filter input \
  ip6 nexthdr { hopopt, routing } drop
```

## RFC 7045 and RFC 7112 Guidance

**RFC 7045** defines expected behavior when processing extension headers:
- Packets with unrecognized hop-by-hop options should be dropped (not silently passed)
- Security devices must not forward packets they cannot fully inspect

**RFC 7112** requires that the first fragment of an IPv6 packet must contain the complete IPv6 header chain up to and including the first upper-layer header (TCP/UDP). This prevents the split-header fragmentation attack.

## Firewall Rules to Mitigate Extension Header Attacks

```bash
# ip6tables: Block known abused extension headers

# Block Routing Header Type 0 (deprecated, enables amplification)
ip6tables -A FORWARD -m rt --rt-type 0 -j DROP

# Block Hop-by-Hop headers on transit traffic (hosts may need this for some protocols)
ip6tables -A FORWARD -m ipv6header --header hop-by-hop -j DROP

# Block excessive fragment headers (atomic fragments)
ip6tables -A INPUT -m frag --fragid 0 --fragmore -j DROP

# Require fragments to contain at least 1280 bytes (except last fragment)
# This is enforced by modern kernels by default
```

## Intrusion Detection for Extension Headers

```bash
# Use Snort/Suricata rules to detect suspicious extension header usage
# Suricata rule: alert on packets with unusual extension header chains
alert ipv6 any any -> any any (
    msg:"Suspicious IPv6 Extension Header Chain";
    ip_proto:43;     # Routing header
    sid:9000001;
)
```

## Summary

IPv6 extension headers can be abused to evade security devices by: splitting transport headers across fragments, exhausting router CPU with Hop-by-Hop headers, or using deprecated Routing Header Type 0 for amplification. Mitigate by blocking RH0, filtering Hop-by-Hop on transit, enforcing RFC 7112 fragment rules, and ensuring your security devices process the full extension header chain before inspecting payloads.
