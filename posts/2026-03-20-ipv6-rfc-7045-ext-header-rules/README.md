# How to Understand RFC 7045 Extension Header Transmission Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RFC 7045, Extension Headers, Standard, Middleboxes

Description: Understand the requirements defined in RFC 7045 for how routers and middleboxes must handle IPv6 extension headers they do not recognize.

## Introduction

RFC 7045 (November 2013) titled "Transmission and Processing of IPv6 Extension Headers" addresses the widespread problem of middleboxes dropping packets with IPv6 extension headers. It clarifies the rules for forwarding nodes (transit routers, firewalls) and provides a framework for managing extension header policies in operational networks.

## Core RFC 7045 Requirements

### For Forwarding Nodes (Routers, Firewalls)

RFC 7045 Section 2.2 states:

```text
If a forwarding node discards a packet because of an unrecognized
extension header, it SHOULD send an appropriate ICMPv6 error message
to the source of the packet (subject to the standard limitations on
ICMPv6 error messages).

A forwarding node MUST NOT drop a packet solely on the basis that
it does not recognize the extension header, unless an explicit
policy permits dropping such packets.
```

Key requirements:
1. **MUST NOT silently drop** packets with unrecognized extension headers
2. **SHOULD send ICMPv6** when dropping is done by explicit policy
3. **Explicit policy required** to justify any extension header drops
4. **SHOULD log** dropped extension header packets for operational visibility

### For IPv6 Nodes (Endpoints)

```text
Destination nodes MUST be able to:
  - Accept extension headers in the recommended order
  - Process extension headers they understand
  - Handle unknown extension headers according to the action bits

  For unknown extension headers (not just unknown options within headers):
  → Process like Next Header = 59 (No Next Header)
  → Ignore the unrecognized header and everything after it
  → (Unless a future RFC changes this behavior)
```

## The IANA Extension Header Registry

RFC 7045 introduced the concept of an IANA registry for extension headers:

```text
IANA registry: "IPv6 Extension Header Types"
https://www.iana.org/assignments/ipv6-parameters/

Currently registered extension headers:
  0   = Hop-by-Hop Options (RFC 8200)
  43  = Routing (RFC 8200)
  44  = Fragment (RFC 8200)
  50  = Encapsulating Security Payload (RFC 4303)
  51  = Authentication Header (RFC 4302)
  59  = No Next Header (RFC 8200)
  60  = Destination Options (RFC 8200)
  135 = Mobility Header (RFC 6275)
  139 = Host Identity Protocol (RFC 7401)
  140 = Shim6 Protocol (RFC 5533)
  253 = Use for experimentation and testing (RFC 3692)
  254 = Use for experimentation and testing (RFC 3692)
```

## RFC 7045 Policy Framework

The RFC introduced a framework for operators to manage extension headers:

```text
Documented policies that forwarding nodes may implement:

1. "Allow" policy:
   Forward all packets with registered extension headers
   (Default for standards-compliant routers)

2. "Explicit block" policy:
   Block specific extension headers with documented justification
   - Must be documented
   - Must be reversible
   - SHOULD generate ICMPv6 on drops

3. "Deprecated" policy:
   Block deprecated extension headers (e.g., RH0)
   - Justified by security concerns
   - Still SHOULD generate ICMPv6

The RFC explicitly discourages the common practice of:
  "Block all unknown extension headers" without documentation
```

## Implementing RFC 7045 Compliance

```bash
#!/bin/bash
# RFC 7045 compliant extension header handling

# 1. Registered extension headers: ALLOW by default

REGISTERED_HEADERS="0 43 44 50 51 59 60 135 139 140"
echo "Allowing registered extension headers: $REGISTERED_HEADERS"

# 2. Deprecated RH0: BLOCK with ICMPv6 (explicit documented policy)
# ip6tables sends ICMPv6 Parameter Problem when using REJECT
sudo ip6tables -A FORWARD -m rt --rt-type 0 \
    -j REJECT --reject-with icmp6-param-prob
# ^ This sends ICMPv6 back to sender (RFC 7045 compliant)

# 3. Unknown extension headers: LOG and FORWARD (RFC 7045 compliant)
# Do NOT drop unknown headers without explicit policy justification
sudo ip6tables -A FORWARD -m ipv6header --header none --soft \
    -j LOG --log-prefix "RFC7045-UNKNOWN-EH: "
# (Note: this rule will rarely match in practice)

# 4. Essential ICMPv6: Always allowed (RFC 4890)
sudo ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 2 -j ACCEPT   # Packet Too Big
sudo ip6tables -A INPUT -p ipv6-icmp --icmpv6-type 3 -j ACCEPT   # Time Exceeded

echo "RFC 7045 compliant configuration applied"
```

## Testing Your Compliance with RFC 7045

```bash
# Test 1: Does your network forward Fragment Headers?
# RFC 7045 says you MUST not drop based solely on extension header type
ping6 -s 1400 -M want <target>

# Test 2: When you drop, do you send ICMPv6?
# Send a packet with RH0 (which should be blocked)
# You should receive ICMPv6 Parameter Problem back

# Test 3: Are your drops documented?
# Review your firewall rules - can you justify each extension header drop?

# Self-assessment checklist:
echo "RFC 7045 Self-Assessment:"
echo "[ ] We allow Fragment Headers (NH=44)"
echo "[ ] We allow IPsec ESP (NH=50) and AH (NH=51)"
echo "[ ] We allow Hop-by-Hop Options (NH=0) for MLD"
echo "[ ] We block RH0 with ICMPv6 response (not silent drop)"
echo "[ ] We have documented policies for any other drops"
echo "[ ] We log dropped extension header packets"
echo "[ ] We do NOT silently drop unknown extension headers"
```

## Conclusion

RFC 7045 provides the standards basis for arguing that middleboxes should not drop IPv6 extension headers without documented policies and ICMPv6 error responses. The core principle is simple: silent drops of unrecognized extension headers violate the RFC and break IPv6 networking. If your operational requirements mandate dropping certain extension headers, you must document the policy, implement it with ICMPv6 responses, and log all affected traffic. This allows sources to diagnose and work around the restriction rather than experiencing mysterious connectivity failures.
