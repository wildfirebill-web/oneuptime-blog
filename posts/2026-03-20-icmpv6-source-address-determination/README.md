# How ICMPv6 Source Address Is Determined

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Source Address, IPv6, RFC 4443, Error Messages

Description: Understand the rules for selecting the source address of ICMPv6 error messages, why the correct source address matters, and how it affects diagnostics and firewall policy.

## Introduction

When a router or host generates an ICMPv6 error message, the choice of source address is not arbitrary — RFC 4443 specifies that the source address must be one of the local addresses assigned to the interface through which the erroneous packet arrived. This rule ensures that the ICMPv6 error message can be routed back to the original source, and that the source address is meaningful for the network that will receive the error.

## RFC 4443 Source Address Rules

```
RFC 4443 Section 2.2: ICMPv6 error message source address selection

Rule 1: If the erroneous packet arrived on an interface with a
        unicast address, use that address as the ICMPv6 source.

Rule 2: Otherwise, select any unicast address assigned to the
        outgoing interface (the interface used to send the ICMPv6).
        Priority order:
        a. Address of same scope as destination of erroneous packet
        b. Global unicast address if destination is global
        c. Link-local address if destination is link-local

Rule 3: Never use:
        - Unspecified address (::) as ICMPv6 source
        - Multicast addresses as ICMPv6 source
        - Loopback address unless error is from/to loopback

Goal: The ICMPv6 error source must be reachable from the
      original packet's source so the error can be delivered.
```

## Why Source Address Matters

```
Impact of ICMPv6 source address on operations:

1. Reachability:
   If ICMPv6 source is a link-local address and the original
   packet's source is on a different network, the link-local
   source cannot be reached → error is undeliverable

2. Firewall policy:
   Firewalls filter ICMPv6 by source address
   Correct source ensures the error passes through the return path
   Wrong source (e.g., internal RFC 1918-like ULA) may be blocked

3. traceroute6 hop identification:
   Each hop sends Time Exceeded from its ingress interface address
   This address identifies the router at that hop
   Choosing wrong source confuses traceroute6 output

4. PMTU Discovery:
   Packet Too Big source is the router's address on the ingress interface
   Source uses this to identify which router reported the bottleneck
```

## Verifying ICMPv6 Source Address Selection

```bash
# Send a packet that will trigger an ICMPv6 error
# Watch what source address the error uses

# Test 1: Trigger Destination Unreachable by sending to an unreachable host
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 1" &
ping6 -c 1 -W 3 2001:db8::99  # Should trigger Dest Unreachable
# Look at the source address of the error message

# Test 2: Trigger Time Exceeded with hop limit = 1
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 3" &
ping6 -t 1 -c 1 8.8.8.8  # First router sends Time Exceeded
# Source of Time Exceeded = router's address on your segment

# Test 3: Trigger Packet Too Big
sudo tcpdump -i eth0 -v "icmp6 and ip6[40] == 2" &
ping6 -M do -s 1452 -c 1 2001:db8::far-server  # If MTU < 1500 in path

# Verify: traceroute6 shows router addresses (ICMPv6 sources at each hop)
traceroute6 2001:db8::server
```

## Simulating Source Address Selection

```python
import socket
import ipaddress

def select_icmpv6_source_address(
    incoming_interface_addrs: list,
    outgoing_interface_addrs: list,
    error_packet_src: str,
) -> str:
    """
    Simulate RFC 4443 ICMPv6 error source address selection.

    Args:
        incoming_interface_addrs: Unicast addresses on incoming interface
        outgoing_interface_addrs: Unicast addresses on outgoing interface
        error_packet_src:         Source address of the erroneous packet

    Returns:
        Selected source address for the ICMPv6 error message
    """
    # Rule 1: If incoming interface has a unicast address, use it
    incoming_unicast = [
        a for a in incoming_interface_addrs
        if not ipaddress.ip_address(a).is_link_local
        and not ipaddress.ip_address(a).is_loopback
        and not ipaddress.ip_address(a).is_multicast
    ]

    if incoming_unicast:
        # Prefer global unicast if error destination is global
        error_src = ipaddress.ip_address(error_packet_src)
        if error_src.is_global:
            global_addrs = [a for a in incoming_unicast
                           if ipaddress.ip_address(a).is_global]
            if global_addrs:
                return global_addrs[0]

        # Fall back to any unicast on incoming interface
        return incoming_unicast[0]

    # Rule 2: Use any unicast on outgoing interface
    if outgoing_interface_addrs:
        return outgoing_interface_addrs[0]

    raise ValueError("No suitable source address found")

# Example: Router with two interfaces
eth0_addrs = ["2001:db8:1::1", "fe80::1"]  # Incoming (LAN side)
eth1_addrs = ["2001:db8:2::1", "fe80::2"]  # Outgoing (WAN side)

src = select_icmpv6_source_address(eth0_addrs, eth1_addrs, "2001:db8:1::100")
print(f"ICMPv6 error source address: {src}")
```

## Common Source Address Mistakes

```
Mistake 1: Using an internal-only source for external errors
  Problem: Error sent from internal address; external firewalls may block it
  Fix:     Use the external interface address for errors to external destinations

Mistake 2: Using :: (unspecified) as ICMPv6 source
  RFC 4443 explicitly prohibits this
  Causes: ICMPv6 cannot be delivered back to source

Mistake 3: Using a loopback source for non-loopback traffic
  RFC 4443 prohibits using loopback unless error is within the loopback context

Mistake 4: Sending ICMPv6 from a different subnet than the incoming interface
  Breaks return routing when the return path is asymmetric
```

## Conclusion

ICMPv6 error message source address selection follows RFC 4443's requirement to use an address from the incoming interface, preferring addresses of the same scope as the destination of the erroneous packet. This ensures errors can be routed back to the original sender. The practical implication for network operators: routers with unnumbered interfaces or point-to-point links must carefully configure which address to use for ICMPv6 error generation, as the wrong choice results in undeliverable errors and makes path debugging much harder.
