# How to Understand IPv6 Fragmentation Attacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Security, Fragmentation, Evasion, Firewall

Description: Learn how IPv6 fragmentation differs from IPv4, how attackers exploit fragment headers to evade security devices, and how to defend against fragmentation-based attacks.

## Overview

IPv6 handles fragmentation differently from IPv4: only the originating host may fragment packets (not routers in transit). This design change, combined with the Fragment Extension Header, creates new attack surfaces that can be exploited to evade intrusion detection, bypass firewalls, and perform denial-of-service attacks.

## IPv6 vs IPv4 Fragmentation

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Who can fragment | Any router | Source host only |
| Fragment field | Always in IP header | Only in Fragment Extension Header |
| Minimum MTU | 576 bytes | 1280 bytes |
| Path MTU Discovery | Optional | Mandatory |
| Atomic fragment | Not applicable | Fragment offset=0, M=0 |

## The IPv6 Fragment Header

When fragmentation is needed, the source adds a Fragment Extension Header:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Next Header  |   Reserved    |      Fragment Offset    |Res|M|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Identification                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Fragment Offset**: offset in 8-byte units
- **M (More Fragments)**: 1 = more fragments follow, 0 = last fragment
- **Identification**: 32-bit ID to reassemble fragments

## Attack Types

### 1. Tiny Fragment Attack

An attacker sends the first fragment so small that the TCP header is split across two fragments. The firewall sees fragment 1 but cannot determine the TCP flags or destination port:

```
Fragment 1: IPv6 + Fragment Header + first 8 bytes of TCP (src/dst port only)
Fragment 2: Remaining TCP header fields (flags, seq numbers) + payload
```

RFC 7112 requires that the first fragment contain the complete upper-layer header — many older devices don't enforce this.

### 2. Overlapping Fragment Attack

Two fragments have overlapping offsets. Different OSes resolve overlaps differently, allowing payload to reach the destination that bypasses IDS inspection:

```
Fragment 1: offset=0, length=64  (contains benign data)
Fragment 2: offset=24 (overlaps with fragment 1) (contains malicious payload)
```

Linux and Windows resolve differently, creating evasion opportunities.

### 3. Atomic Fragment Attack (RFC 8021)

An attacker sends a Router Advertisement with MTU < 1280, causing the host to generate "atomic fragments" (Fragment Header with offset=0 and M=0). These can be spoofed to fragment otherwise non-fragmented communications:

```bash
# Detect atomic fragments with tcpdump
tcpdump -i eth0 'ip6[6]==44 and (ip6[42:2] & 0xfff9) == 0'
# ip6[6]==44 = Fragment Header
# offset=0 and M=0 = atomic fragment
```

RFC 8021 documents the atomic fragment vulnerability and recommends never generating them.

### 4. Resource Exhaustion via Incomplete Fragment Sets

An attacker sends first fragments without ever sending the last fragment. The victim must hold partial reassembly buffers:

```
Attacker → sends fragment 1 (M=1) with random ID
Attacker → never sends fragment 2
Victim → holds reassembly buffer for timeout period (~60 seconds)
Repeat → exhaust kernel memory
```

## Detection

### tcpdump Detection Rules

```bash
# Capture all IPv6 fragmented traffic
tcpdump -i eth0 'ip6[6]==44'

# Capture first fragments only (offset=0, M=1)
tcpdump -i eth0 'ip6[6]==44 and (ip6[42:2] & 0x0001) == 1 and (ip6[42:2] >> 3) == 0'

# Detect non-initial fragments (offset > 0)
tcpdump -i eth0 'ip6[6]==44 and (ip6[42:2] >> 3) > 0'
```

### Suricata/Snort Rules

```
# Alert on IPv6 fragments
alert ipv6 any any -> any any (
    msg:"IPv6 Fragment Header Detected";
    ip_proto:44;
    sid:9000010;
    rev:1;
)

# Alert on tiny first fragments (potential header split)
alert ipv6 any any -> any any (
    msg:"IPv6 Tiny Fragment - Possible Evasion";
    ip_proto:44;
    dsize:<48;
    sid:9000011;
    rev:1;
)
```

## Firewall Mitigation

```bash
# ip6tables: Drop non-initial fragments (cannot inspect transport headers)
ip6tables -A FORWARD -m frag --fragid 0 --fragmore -j DROP

# Drop fragments where TCP header might be split across fragments
# Require first fragment to be large enough to contain full transport header
ip6tables -A INPUT -m frag --fragfirst --fragmore -m length --length 0:1279 -j DROP

# nftables: Block all fragmented IPv6 traffic
nft add rule ip6 filter input frag exists drop

# Or be selective — block forwarded fragments only
nft add rule ip6 filter forward frag exists drop
```

## RFC 7112: Requiring Complete Header Chain in First Fragment

RFC 7112 (2014) requires that the first fragment of an IPv6 packet contain the complete extension header chain up to (and including) the first upper-layer protocol header:

```
First Fragment MUST contain:
  IPv6 Header
  → All Extension Headers (Hop-by-Hop, Routing, etc.)
  → Fragment Header
  → At minimum the first upper-layer header (TCP/UDP/ICMPv6)
```

Security devices SHOULD drop first fragments that don't satisfy this requirement.

```bash
# Test if your device enforces RFC 7112
# Send a crafted tiny first fragment and see if it's dropped
hping3 -6 --ipproto 44 -d 8 <target>   # Requires hping3 with IPv6 support
```

## Kernel Tuning for Fragment Reassembly Limits

On Linux, you can limit fragment reassembly memory to reduce DoS impact:

```bash
# View current fragment settings
sysctl net.ipv6.ip6frag_high_thresh
sysctl net.ipv6.ip6frag_low_thresh
sysctl net.ipv6.ip6frag_time

# Reduce reassembly timeout (default 60s, reduce to 10s)
sysctl -w net.ipv6.ip6frag_time=10

# Reduce reassembly memory (default ~4MB)
sysctl -w net.ipv6.ip6frag_high_thresh=2097152
sysctl -w net.ipv6.ip6frag_low_thresh=1048576
```

## Summary

IPv6 fragmentation attacks exploit the Fragment Extension Header to split transport headers across fragments, evade security inspection, or exhaust reassembly buffers. Defend by enforcing RFC 7112 (first fragment must contain complete upper-layer header), dropping non-initial fragments at security boundaries, applying kernel limits on reassembly memory, and alerting on unusual fragment patterns with IDS rules.
