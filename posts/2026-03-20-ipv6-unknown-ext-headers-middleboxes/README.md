# How to Handle Unknown Extension Headers in Middleboxes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Extension Headers, Middleboxes, Firewall, RFC 7045

Description: Understand how middleboxes should handle unknown IPv6 extension headers, the RFC 7045 guidelines, and why incorrect handling causes connectivity failures.

## Introduction

A "middlebox" is any network device between two communicating endpoints that is not the final destination - firewalls, load balancers, deep packet inspection systems, and network address translators. The handling of unknown IPv6 extension headers by middleboxes is a major source of connectivity problems in IPv6 networks. RFC 7045 defines clear rules for how middleboxes MUST handle extension headers.

## The Problem: Middlebox Drop Behavior

Many IPv6 deployments suffer from "extension header filtering" where middleboxes incorrectly drop packets containing extension headers they don't recognize. This breaks:

- IPv6 fragmentation (Fragment Header = 44)
- IPsec (AH = 51, ESP = 50)
- Mobility (MIPv6 = 135)
- New protocols that add new extension headers

Research has shown that a significant fraction of internet paths drop packets with extension headers:

```text
Measured extension header drop rates (circa 2015-2020 studies):
  Fragment Header (44): 30-50% of paths drop these packets
  Routing Header (43):  40-60% of paths drop these packets
  Hop-by-Hop (0):      50-70% of paths drop these packets
  AH (51):             40-50% of paths drop these packets
```

## RFC 7045: Extension Header Transmission Rules

RFC 7045 defines what middleboxes MUST and MUST NOT do:

```text
MUST forward (cannot drop based on extension header type alone):
  - Fragment Header (44): Required for fragmentation to work
  - Authentication Header (51): Required for IPsec
  - ESP Header (50): Required for IPsec
  - Destination Options (60): Common for IPv6 features
  - Hop-by-Hop Options (0): Required for MLD and other protocols

MAY drop based on policy (if explicit operator policy):
  - Routing Header Type 0 (deprecated, security risk)
  - Unknown extension headers only if operator policy requires it
  - Must log the drops and ideally send ICMPv6 back

MUST NOT silently drop:
  - Packets with ANY extension header without explicit policy reason
  - Packets with only because of an unknown extension header type
```

## Testing Extension Header Passthrough

```bash
# Test if a path passes Fragment Headers

# Create a fragmented ping6 (forces use of Fragment Header)
ping6 -s 1280 -M want 2001:db8::target  # Force fragmentation

# Test with specific extension headers using scapy
python3 << 'EOF'
from scapy.all import *

# Test Fragment Header passthrough
pkt = IPv6(dst="2001:db8::target") / \
      IPv6ExtHdrFragment(id=0x1234, m=0, offset=0) / \
      UDP(sport=12345, dport=53) / DNS(qd=DNSQR(qname="test.example.com"))

# Send and see if we get a response
resp = sr1(pkt, timeout=2, verbose=0)
if resp:
    print("Fragment Header: PASSED")
else:
    print("Fragment Header: DROPPED (or no route)")
EOF

# Test ICMPv6 with Hop-by-Hop Router Alert
python3 << 'EOF'
from scapy.all import *

# MLD Membership Query uses Router Alert
pkt = IPv6(dst="ff02::1", hlim=1) / \
      IPv6ExtHdrHopByHop(options=[RouterAlert(value=0)]) / \
      ICMPv6MLQuery()
send(pkt, verbose=0)
print("Sent MLD query with Router Alert")
EOF
```

## Firewall Configuration for Extension Header Passthrough

```bash
# ip6tables: allow essential extension headers

# Allow fragmented packets (Fragment Header)
sudo ip6tables -A FORWARD -m frag --fragmore -j ACCEPT
sudo ip6tables -A FORWARD -m frag --fraglast -j ACCEPT
sudo ip6tables -A FORWARD -m frag --fragfirst -j ACCEPT

# Allow IPsec (AH and ESP)
sudo ip6tables -A FORWARD -p ah -j ACCEPT
sudo ip6tables -A FORWARD -p esp -j ACCEPT

# Allow Hop-by-Hop options (needed for MLD)
sudo ip6tables -A FORWARD -m ipv6header --header hop -j ACCEPT

# Allow Destination Options
sudo ip6tables -A FORWARD -m ipv6header --header dst -j ACCEPT

# Log but do NOT drop Routing Headers (except RH0)
sudo ip6tables -A FORWARD -m rt --rt-type 0 -j LOG --log-prefix "RH0-DROP: "
sudo ip6tables -A FORWARD -m rt --rt-type 0 -j DROP  # Only RH0 is dangerous
```

## nftables Configuration

```text
# /etc/nftables.conf - extension header handling
table inet filter {
    chain forward {
        type filter hook forward priority 0; policy drop;

        # Allow established/related
        ct state established,related accept

        # Allow fragment headers (required for IPv6 fragmentation)
        ip6 nexthdr 44 accept

        # Allow IPsec
        ip6 nexthdr 50 accept  # ESP
        ip6 nexthdr 51 accept  # AH

        # Allow Hop-by-Hop (required for MLD)
        ip6 nexthdr 0 accept

        # Drop deprecated RH0 (security risk)
        ip6 nexthdr 43 ip6 rt type 0 drop

        # Log unknown extension headers but forward
        ip6 nexthdr != { 6, 17, 58, 44, 50, 51, 0, 43, 60, 135 } log prefix "Unknown EH: "
        accept
    }
}
```

## Conclusion

Middleboxes that indiscriminately drop IPv6 packets containing unknown extension headers are a significant barrier to IPv6 adoption and feature development. RFC 7045 provides clear guidance: forward extension headers you don't understand unless you have an explicit policy reason to drop them, and log drops with details. The only unambiguously safe drop policy is for Routing Header Type 0 (deprecated for security). Properly configured firewalls allow fragmentation, IPsec, and Hop-by-Hop options while blocking only demonstrably dangerous extension header types.
