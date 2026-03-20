# How to Understand RFC 9098 Operational Implications of Extension Headers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RFC 9098, Extension Headers, Operations, Networking

Description: Understand the operational implications of IPv6 extension headers documented in RFC 9098, including deployment challenges, measurement data, and operator recommendations.

## Introduction

RFC 9098 (August 2021) "Operational Implications of IPv6 Packets with Extension Headers" is the definitive operational document on IPv6 extension header behavior in real-world networks. It documents measured drop rates, explains why these drops occur, and provides recommendations for operators and protocol designers. Understanding RFC 9098 helps operators make informed decisions about extension header policies.

## Key Findings of RFC 9098

RFC 9098 documents the following operational realities:

```
1. Fragment Header (NH=44) Drop Rates:
   - ~30-50% of network paths drop packets with Fragment Headers
   - Cause: Security policies preventing fragment-based attacks
   - Impact: Breaks IPv6 Path MTU Discovery
   - Recommendation: Operators should allow fragmented IPv6 packets
                     or explicitly configure adequate MTU throughout

2. Routing Header (NH=43) Drop Rates:
   - High drop rates, especially for unknown routing header types
   - Type 0 (deprecated): Should be dropped - security risk
   - Types 2, 3, 4: Should be forwarded

3. Hop-by-Hop Options (NH=0) Slow-Path Processing:
   - Virtually all modern routers punt HbH packets to the CPU
   - This creates denial-of-service risk
   - Recommendation: Avoid HbH in new protocols where possible
   - Critical exception: MLD MUST use HbH Router Alert

4. Unknown Extension Headers:
   - Many devices drop unknown extension headers silently
   - This prevents deployment of new IPv6 extensions
   - RFC 9098 recommends allowing unknown but registered headers
```

## The Hop-by-Hop CPU Vulnerability

RFC 9098 provides detailed analysis of the Hop-by-Hop CPU problem:

```
Why HbH causes CPU exhaustion:

1. Specification says: All routers MUST examine Hop-by-Hop options
2. Hardware: Cannot implement arbitrary option processing in ASICs
3. Result: Any packet with HbH header → software slow path → CPU
4. Attack: Send millions of packets with HbH headers to a router
5. Result: Router CPU saturated, legitimate traffic drops

This is why RFC 9098 Section 2.1 notes that some routers
completely drop packets with Hop-by-Hop options as a DoS defense.

Affected protocols that use Hop-by-Hop:
  - MLD (Multicast Listener Discovery) - uses Router Alert in HbH
  - RSVP - uses Router Alert in HbH
  - Jumbograms - use Jumbo Payload option in HbH
```

## Operational Recommendations from RFC 9098

```python
# Summarize RFC 9098 recommendations for operators
RFC9098_RECOMMENDATIONS = {
    "Hop-by-Hop (NH=0)": {
        "action": "Rate-limit incoming, allow for MLD",
        "reason": "HbH forces slow-path CPU processing; DoS risk",
        "exception": "MLD queries/reports require Router Alert in HbH",
        "recommended_rate_limit": "~100 pps per source"
    },
    "Fragment (NH=44)": {
        "action": "Allow, but implement anti-spoofing",
        "reason": "Required for Path MTU Discovery to function",
        "note": "Use ip6tables frag module to inspect fragment headers",
        "security": "Block overlapping fragments (fragment overlap attacks)"
    },
    "Routing (NH=43) Type 0": {
        "action": "Block with ICMPv6",
        "reason": "Deprecated security vulnerability (RFC 5095)",
    },
    "Routing (NH=43) Types 2/3/4": {
        "action": "Allow",
        "reason": "MIPv6 (Type 2), RPL (Type 3), SRv6 (Type 4) are in use",
    },
    "ESP (NH=50)": {
        "action": "Allow",
        "reason": "Required for IPsec VPN deployments",
    },
    "AH (NH=51)": {
        "action": "Allow",
        "reason": "Required for IPsec authentication",
    },
    "Unknown extension headers": {
        "action": "Log and forward (per RFC 7045)",
        "reason": "Silent drops prevent future extension header deployment",
    }
}

for header, rec in RFC9098_RECOMMENDATIONS.items():
    print(f"\n{header}:")
    for key, value in rec.items():
        print(f"  {key}: {value}")
```

## Fragment Header and Path MTU Discovery

RFC 9098 explains the interconnection between Fragment Header drops and Path MTU Discovery:

```
Normal Path MTU Discovery (RFC 8201):
1. Source sends packet > Path MTU
2. Router drops packet, sends ICMPv6 "Packet Too Big"
3. Source reduces packet size, no fragmentation needed

When ICMPv6 "Packet Too Big" is blocked (firewall misconfiguration):
  → Source never learns about smaller MTU
  → Source keeps sending oversized packets
  → All packets are silently dropped at the bottleneck
  → Connection "black hole" (appears to work for small packets, fails for large)

RFC 9098 recommendation:
  → NEVER block ICMPv6 Packet Too Big messages (type 2)
  → ICMPv6 filtering should follow RFC 4890 (essential ICMPv6 must be allowed)
```

## Measuring Extension Header Impact

```bash
# RFC 9098 compliant measurement methodology

# 1. Establish baseline reachability (no extension headers)
ping6 -c 20 -s 56 target.example.com
BASELINE_LOSS=$?

# 2. Test fragment header passthrough
ping6 -c 20 -s 1400 -M want target.example.com
FRAGMENT_LOSS=$?

# 3. Compare (if FRAGMENT_LOSS > BASELINE_LOSS, Fragment Headers are dropped)
echo "Baseline loss: $BASELINE_LOSS"
echo "Fragment loss: $FRAGMENT_LOSS"
if [ $FRAGMENT_LOSS -gt $BASELINE_LOSS ]; then
    echo "WARNING: Fragment Headers may be dropped on this path"
fi

# 4. Test ICMPv6 Packet Too Big (check if MTU discovery works)
# Send a large packet and see if you get PTB back
sudo tcpdump -i eth0 "icmp6 and ip6[40] == 2" &
ping6 -s 1450 -M dont target.example.com  # Oversized, check for PTB
sleep 3
kill %1
```

## Conclusion

RFC 9098 provides operators with measurement data and recommendations for managing IPv6 extension headers in production networks. The key insights: Hop-by-Hop Options create CPU exhaustion risk and should be rate-limited, Fragment Headers are frequently dropped but must be allowed for Path MTU Discovery, and ICMPv6 Packet Too Big messages must never be filtered. Protocol designers should consult RFC 9098 before using extension headers in new specifications, as the deployment reality may make reliable extension header delivery impossible on many internet paths.
