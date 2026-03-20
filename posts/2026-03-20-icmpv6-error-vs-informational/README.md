# How to Understand ICMPv6 Error vs Informational Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Error Messages, Informational Messages, IPv6, RFC 4443

Description: Understand the distinction between ICMPv6 error messages (Types 1-127) and informational messages (Types 128-255), their different rules and behaviors, and why this classification matters for...

## Introduction

RFC 4443 divides ICMPv6 messages into two classes: error messages (Type values 0-127) and informational messages (Type values 128-255). This classification is not just organizational - it carries specific behavioral rules. Error messages must include a copy of the offending packet, are subject to rate limiting, must not be generated in response to other ICMPv6 error messages, and require different firewall treatment than informational messages.

## Error Messages (Types 0-127)

```text
ICMPv6 Error Message rules (RFC 4443):

1. Generated in response to a problem processing an IPv6 packet
2. MUST include as much of the offending packet as possible
   (without exceeding 1280 bytes total ICMPv6 message size)
3. MUST NOT be sent in response to:
   - An ICMPv6 error message (prevents error loops)
   - A packet sent to a multicast address (except for Packet Too Big
     and Parameter Problem with code 2)
   - A packet that appears to have a forged source (e.g., unspecified
     address :: as source)
4. SHOULD be rate-limited to prevent flooding
5. Type range: 0-127

Current error messages:
  Type 1: Destination Unreachable (Codes 0-7)
  Type 2: Packet Too Big
  Type 3: Time Exceeded (Codes 0-1)
  Type 4: Parameter Problem (Codes 0-2)
```

## Informational Messages (Types 128-255)

```text
ICMPv6 Informational Message characteristics:

1. Not generated in response to errors
2. Do not include a copy of an offending packet
3. Subject to their own specific rules
4. Type range: 128-255

Current informational messages:
  Type 128: Echo Request
  Type 129: Echo Reply
  Type 130: Multicast Listener Query (MLD)
  Type 131: Multicast Listener Report (MLD)
  Type 132: Multicast Listener Done (MLD)
  Type 133: Router Solicitation (NDP)
  Type 134: Router Advertisement (NDP)
  Type 135: Neighbor Solicitation (NDP)
  Type 136: Neighbor Advertisement (NDP)
  Type 137: Redirect
  Type 143: MLDv2 Multicast Listener Report
```

## Why the Distinction Matters for Firewalls

```text
Firewall implications of error vs informational classification:

Error messages:
  MUST allow: Packet Too Big (Type 2) - essential for PMTUD
  MUST allow: Destination Unreachable (Type 1) - needed for error reporting
  SHOULD allow: Time Exceeded (Type 3) - needed for traceroute6
  SHOULD allow: Parameter Problem (Type 4) - needed for error reporting
  Blocking Type 2 BREAKS PMTUD for all TCP connections

Informational messages:
  NDP types (133-137) MUST be allowed on local link (link-local scope)
  MLD types (130-132, 143) MUST be allowed for multicast to work
  Echo (128-129) can be blocked if ping6 not needed (but helpful to allow)
  NDP on external interfaces should be blocked (not valid cross-router)
```

## Classification Check in Code

```python
def classify_icmpv6(icmp_type: int) -> dict:
    """
    Classify an ICMPv6 message type and provide firewall guidance.
    """
    error_messages = {
        1:  ("Destination Unreachable", "allow"),
        2:  ("Packet Too Big",          "MUST allow - breaks PMTUD if blocked"),
        3:  ("Time Exceeded",           "allow - needed for traceroute6"),
        4:  ("Parameter Problem",       "allow"),
    }

    informational_messages = {
        128: ("Echo Request",            "allow if ping6 desired"),
        129: ("Echo Reply",              "allow if ping6 desired"),
        130: ("MLD Query",               "allow on local segments"),
        131: ("MLD Report",              "allow on local segments"),
        132: ("MLD Done",                "allow on local segments"),
        133: ("Router Solicitation",     "allow link-local; block on transit"),
        134: ("Router Advertisement",    "allow link-local; block on transit"),
        135: ("Neighbor Solicitation",   "allow link-local; block on transit"),
        136: ("Neighbor Advertisement",  "allow link-local; block on transit"),
        137: ("Redirect",                "allow link-local; block on transit"),
        143: ("MLDv2 Report",            "allow on local segments"),
    }

    if icmp_type in error_messages:
        name, guidance = error_messages[icmp_type]
        return {"class": "error", "name": name, "guidance": guidance}
    elif icmp_type in informational_messages:
        name, guidance = informational_messages[icmp_type]
        return {"class": "informational", "name": name, "guidance": guidance}
    elif icmp_type < 128:
        return {"class": "error", "name": f"Unknown error type {icmp_type}", "guidance": "evaluate"}
    else:
        return {"class": "informational", "name": f"Unknown informational type {icmp_type}", "guidance": "evaluate"}

# Test all known types

for t in [1, 2, 3, 4, 128, 129, 133, 134, 135, 136]:
    r = classify_icmpv6(t)
    print(f"Type {t:3d} ({r['class']:15s}): {r['name']:<30} → {r['guidance']}")
```

## Rate Limiting Error Messages

```bash
# Linux automatically rate-limits ICMPv6 error generation
# Check current rate limit settings
cat /proc/sys/net/ipv6/icmp/ratelimit
# Default: 1000 (milliseconds between ICMPv6 error generation)

# Check rate limit burst size
cat /proc/sys/net/ipv6/icmp/ratemask
# Bitmask of message types subject to rate limiting

# View ICMPv6 statistics (includes error generation counts)
cat /proc/net/snmp6 | grep -i icmp
```

## Conclusion

The error/informational classification of ICMPv6 messages has practical implications for both firewall policy and protocol implementation. Error messages (Types 1-127) trigger specific behaviors: rate limiting, anti-loop rules, and inclusion of the offending packet. Informational messages (Types 128-255) include NDP and MLD, which are essential for IPv6 to function at all on a network segment. The most critical rule for firewall administrators: never block ICMPv6 Type 2 (Packet Too Big), as this breaks Path MTU Discovery for all IPv6 TCP connections.
