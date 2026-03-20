# How to Understand ICMPv6 Redirect Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Redirect, NDP, IPv6, Router, RFC 4861

Description: Understand ICMPv6 Redirect messages (Type 137), when routers send them, how hosts update their destination cache, and when to block redirects for security.

## Introduction

ICMPv6 Redirect (Type 137) is sent by a router to inform a host that a better first-hop exists for a particular destination. When a host sends a packet to a router, and that router knows a better route via another on-link router, it forwards the packet but also sends a Redirect to the host. The host updates its destination cache to use the better next-hop for future packets to that destination.

## Redirect Message Format

```text
ICMPv6 Redirect (Type 137):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type=137  |   Code = 0    |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Reserved                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                         Target Address                        +
|                (the better next-hop address)    (128 bits)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                      Destination Address                      +
|               (the destination of the original packet)        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Options: Target Link-Layer Address and Redirected Header    |

IPv6 Header:
  Source:      Router's link-local address (fe80::...)
  Destination: Source of the original packet (host being redirected)
  Hop Limit:   255 (mandatory)
```

## When Redirects Are Sent

```text
Redirect conditions (RFC 4861 Section 8.2):

A router MUST send a Redirect when ALL of the following are true:

1. The packet being forwarded arrived on the SAME interface it will
   be forwarded on (both ingress and egress are the same interface)
   → This means both the sending host and the better next-hop
     are on the same link

2. The better next-hop address (Target) is different from the
   Destination address in the original packet
   → Exception: if the Destination IS the better next-hop (host
     is directly on-link), Target = Destination

3. The packet is not multicast

4. The router is not forwarding packets to the host via its own
   address (i.e., the host's traffic arrives from the link where
   the better route also exists)

Example scenario:
  Host: 2001:db8::100 (gateway = Router-A fe80::1)
  Router-A has: traffic to 2001:db8:2::/64 goes via Router-B (fe80::2)
  Router-B is on the same link as Host

  Flow:
  1. Host sends to 2001:db8:2::1, via Router-A
  2. Router-A forwards to Router-B (same link)
  3. Router-A sends Redirect to Host: "use fe80::2 for 2001:db8:2::/64"
  4. Host caches: dest 2001:db8:2::1 → next-hop fe80::2
  5. Future packets skip Router-A and go directly to Router-B
```

## Processing Redirects

```bash
# Check the IPv6 destination cache (shows cached redirects)

ip -6 route show cache | grep redirect

# View all redirect entries (next-hop is different from default GW)
ip -6 route show cache

# Example: a redirect entry shows an alternative next-hop
# 2001:db8:2::1 via fe80::2 dev eth0 src 2001:db8::100
#   cache  expires 60sec redirect

# Accept redirects from routers (default behavior)
cat /proc/sys/net/ipv6/conf/eth0/accept_redirects
# 1 = accept (default on hosts)

# Disable accepting redirects (useful on routers)
sudo sysctl -w net.ipv6.conf.all.accept_redirects=0
sudo sysctl -w net.ipv6.conf.default.accept_redirects=0

# Enable on hosts (restoring default)
sudo sysctl -w net.ipv6.conf.all.accept_redirects=1
```

## Security Considerations for Redirects

```bash
# Redirects can be used for traffic hijacking:
# 1. Attacker on the same link sends forged Redirect to host
# 2. Redirect says: "use attacker's address for destination X"
# 3. Host routes all traffic to X through attacker

# Defense 1: Accept redirects only from current default gateway
# Linux checks if redirect source is in default router list

# Defense 2: Disable redirects on hosts that don't need them
sudo sysctl -w net.ipv6.conf.eth0.accept_redirects=0

# Defense 3: Block Redirect messages from untrusted sources in ip6tables
DEFAULT_GW=$(ip -6 route show default | awk '/via/ {print $3; exit}')
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type redirect \
    -s "$DEFAULT_GW" -j ACCEPT
sudo ip6tables -A INPUT -p icmpv6 --icmpv6-type redirect -j DROP

# Defense 4: ND Inspection on switch (validates Redirect source must be
# a router that previously sent Router Advertisements)
```

## Conclusion

ICMPv6 Redirect is a network optimization mechanism that allows routers to inform hosts about better first-hop paths on the same link. The Hop Limit 255 requirement ensures redirects can only come from local-link nodes. The destination cache stores redirect-based routes separately from route-table entries, with shorter expiry times. From a security perspective, Redirect messages should be accepted only from the current default gateway, and on hosts where routing optimization is not needed, redirects can be disabled entirely with `net.ipv6.conf.*.accept_redirects=0`.
