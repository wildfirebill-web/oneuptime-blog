# How to Understand the SRv6 SID Address Space (5f00::/16)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, SRv6, 5f00, RFC 9602, Segment Routing, SID

Description: Understand the SRv6 SID address space 5f00::/16 allocated by RFC 9602, its properties, and how operators should plan and use SRv6 SID allocations.

## Introduction

RFC 9602 (2024) formally allocates `5f00::/16` as the globally routable address space for SRv6 Segment Identifiers (SIDs). This is a 16-bit prefix, providing an enormous address space for SRv6 deployments worldwide. Unlike earlier SRv6 deployments that used arbitrary IPv6 addresses, using `5f00::/16` provides interoperability and clear identification.

## Key Properties

| Property | Value |
|---|---|
| Prefix | 5f00::/16 |
| RFC | RFC 9602 (2024) |
| Source | True |
| Destination | True |
| Forwardable | True |
| Globally Reachable | True |

## Address Space Scale

```python
import ipaddress

SRV6_SPACE = ipaddress.IPv6Network("5f00::/16")

# Calculate sub-allocations available

print(f"Total /48 locators available: {2**(48-16):,}")
# 4,294,967,296 - over 4 billion /48 node locators

print(f"Total /128 SIDs: {2**(128-16):,}")
# An astronomically large number

# Typical allocation:
# /16 global SRv6 space: 5f00::/16
# /32 per-AS block: 5f00:asn::/32
# /48 per-node locator: 5f00:asn:node::/48
# /128 per-SID: 5f00:asn:node:function::

example_allocations = {
    "AS65001 block":    "5f00:fe81::/32",   # ASN 65001 = 0xFE81
    "Node R1 locator":  "5f00:fe81:1::/48",
    "Node R2 locator":  "5f00:fe81:2::/48",
    "R1 End SID":       "5f00:fe81:1:0001::/128",
    "R1 End.X SID":     "5f00:fe81:1:e001::/128",
    "R1 End.DT6 SID":   "5f00:fe81:1:e002::/128",
}
for name, prefix in example_allocations.items():
    print(f"{name}: {prefix}")
```

## Routing 5f00::/16

```bash
# 5f00::/16 is globally routable via BGP
# Each operator advertises their sub-prefix

# FRR: advertise your SRv6 locator block via BGP
router bgp 65001
  address-family ipv6 unicast
    network 5f00:fe81::/32  ! Your AS block
  !
!

# Each node advertises its /48 locator via IS-IS or OSPF
router isis CORE
  segment-routing srv6
    locator MAIN
      prefix 5f00:fe81:1::/48
```

## Filtering in BGP

```bash
# Accept SRv6 locators from trusted peers only
# Do not accept 5f00::/16 more-specifics from customers
# (they should not advertise SRv6 SIDs)

# FRR route map
ip prefix-list DENY_SRV6_SPACE seq 5 deny 5f00::/16 le 128
ip prefix-list DENY_SRV6_SPACE seq 100 permit ::/0 le 128

# Apply to customer BGP sessions
router bgp 65001
  neighbor 2001:db8::customer route-map DENY_SRV6_SPACE in
```

## Verifying 5f00::/16 Routing

```bash
# Check if your SRv6 locator is reachable from the internet
# (requires 5f00::/16 to be routed in your BGP)
traceroute6 5f00:fe81:2::  # Should reach Node R2

# Check BGP advertisement
show bgp ipv6 unicast 5f00:fe81::/32

# Verify SID is programmed in FIB
ip -6 route show 5f00:fe81:1:e001::/128
# Should show: encap seg6local action End.X ...
```

## Conclusion

The `5f00::/16` allocation makes SRv6 SIDs globally routable and clearly distinguishable. Operators should obtain a sub-prefix from their RIR or use their ASN to carve a deterministic block. Structure allocations hierarchically: /32 per AS, /48 per node, /128 per SID function. Monitor SRv6 locator reachability from multiple vantage points using OneUptime to ensure your SRv6 infrastructure is globally visible.
