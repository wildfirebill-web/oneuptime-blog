# How to Understand the AMT Address Space (2001:3::/32)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, AMT, Automatic Multicast Tunneling, 2001:3::/32, RFC 7450, Multicast

Description: Understand the AMT (Automatic Multicast Tunneling) address space 2001:3::/32 (RFC 7450), how AMT relays use it, and its role in enabling IPv6 multicast across unicast networks.

## Introduction

`2001:3::/32` is allocated for Automatic Multicast Tunneling (AMT) as defined in RFC 7450. AMT allows IPv6 multicast receivers behind unicast-only networks to receive multicast streams by tunneling through an AMT gateway/relay pair. Unlike other special-purpose ranges, `2001:3::/32` is both forwardable and globally reachable.

## Key Properties

| Property | Value |
|---|---|
| Prefix | 2001:3::/32 |
| RFC | RFC 7450 |
| Source | True |
| Destination | True |
| Forwardable | Yes |
| Globally Reachable | Yes |

## How AMT Works

```
Architecture:
  Multicast Source → Multicast Network → AMT Relay
                                              ↕ (UDP tunnel)
                              AMT Gateway ← Unicast Network
                                    ↕
                              Multicast Receiver

AMT Relay: Connected to multicast network, accessible via unicast
AMT Gateway: On the receiver's unicast network, discovers relay via DNS

DNS discovery:
  _amt._udp.example.com → SRV → amt-relay.example.com
  amt-relay.example.com → AAAA → 2001:3::relay-address
```

## AMT Relay Address in 2001:3::/32

```python
import ipaddress

AMT_BLOCK = ipaddress.IPv6Network("2001:3::/32")

def is_amt_relay(addr_str: str) -> bool:
    """Check if an IPv6 address is in the AMT relay space."""
    try:
        return ipaddress.IPv6Address(addr_str) in AMT_BLOCK
    except ValueError:
        return False

# AMT relay addresses are assigned within 2001:3::/32
# Typically by content providers or ISPs offering multicast

# Example AMT relay assignment
example_relay = "2001:3::1"
print(f"Is AMT relay space: {is_amt_relay(example_relay)}")  # True

# AMT gateway discovers relay via DNS SRV record
# _amt._udp.multicast.example.com IN SRV 0 0 2268 amt-relay.example.com
# amt-relay.example.com IN AAAA 2001:3::relay1
```

## AMT Protocol Exchange

```
1. Gateway sends Relay Discovery (UDP 2268)
   → dst: AMT anycast address or known relay
   → src: Gateway address

2. Relay responds with Relay Advertisement
   ← src: Relay IPv6 address (in 2001:3::/32)
   ← Contains: relay's unicast address

3. Gateway sends Request
   → dst: Relay unicast address

4. Relay sends Membership Query (tunneled IGMP/MLD)
   ← Relay checks what groups gateway wants

5. Gateway sends Membership Update (MLD Report)
   → Requests specific multicast group

6. Relay sends multicast data encapsulated in UDP
   → Gateway decapsulates and delivers locally
```

## Filtering AMT in Firewall

```bash
# Allow AMT relay traffic (UDP 2268)
ip6tables -A INPUT -p udp --dport 2268 -s 2001:3::/32 -j ACCEPT
ip6tables -A OUTPUT -p udp --dport 2268 -d 2001:3::/32 -j ACCEPT

# If you don't use AMT, block it
ip6tables -A INPUT -s 2001:3::/32 -j DROP
ip6tables -A OUTPUT -d 2001:3::/32 -j DROP
ip6tables -A INPUT -p udp --dport 2268 -j DROP
```

## AMT vs PIM-SM for Multicast

```
Native Multicast (PIM-SM):
  - Requires multicast-enabled routers throughout the path
  - Best for intranet multicast
  - No tunneling overhead

AMT:
  - Works across unicast-only networks (internet)
  - Uses UDP tunneling (overhead)
  - Essential for "over-the-top" multicast (OTT streaming)
  - Content providers use AMT for live streaming to ISPs
    without multicast peering
```

## Conclusion

The `2001:3::/32` AMT space enables IPv6 multicast delivery across unicast networks. AMT relays use addresses in this block and are globally reachable. If your network does not run AMT, block `2001:3::/32` and UDP port 2268 at your firewall. Monitor AMT relay availability with OneUptime if your organization depends on AMT for multicast content delivery.
