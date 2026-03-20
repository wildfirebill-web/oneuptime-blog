# How to Understand ICMPv6 Message Processing Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ICMPv6, Message Processing, RFC 4443, IPv6, Protocol Rules

Description: Understand the rules governing when ICMPv6 messages must and must not be generated, how to validate incoming messages, and the anti-loop rules that prevent cascading errors.

## Introduction

RFC 4443 defines strict rules for when ICMPv6 error messages may and may not be generated. These rules prevent error storms (an ICMPv6 error triggering another ICMPv6 error, ad infinitum), reduce the risk of spoofing attacks, and ensure that ICMPv6 errors are only sent when they provide useful information. Every IPv6 implementation must follow these rules exactly.

## Rules for When NOT to Generate ICMPv6 Errors

```text
RFC 4443 Section 2.4: Do NOT send ICMPv6 errors in response to:

1. ICMPv6 error messages
   → Error in response to error = infinite loop
   → Easy check: Type < 128 (error) responding to another Type < 128

2. Packets sent to a multicast or anycast address
   → Exception: Packet Too Big (Type 2) may be sent for multicast
   → Exception: Parameter Problem (Type 4, Code 2) for unrecognized option
   → Reason: One error message would not reach all senders

3. Packets sent as a link-layer multicast or broadcast
   → Same reason: many senders, one error response

4. Packets whose source address does not uniquely identify a single node:
   → Unspecified address (::) as source
   → Multicast source address
   → Anycast source address (ambiguous)

5. Packets sent to a loopback, unspecified, or anycast DESTINATION
   (when the erroneous packet's source would be unreachable)
```

## Rules for When TO Generate ICMPv6 Errors

```text
RFC 4443: Generate ICMPv6 errors when:

1. A packet CANNOT BE FORWARDED due to routing or MTU issues
   → Destination Unreachable (Type 1, Code 0 or 3)
   → Packet Too Big (Type 2)

2. A packet EXCEEDS HOP LIMIT
   → Time Exceeded (Type 3, Code 0)
   → Always sent by the router that decremented to 0

3. A packet CONTAINS AN INVALID HEADER FIELD
   → Parameter Problem (Type 4, Code 0)
   → Pointer must identify the problematic byte

4. Fragment reassembly timer expires with incomplete fragments
   → Time Exceeded (Type 3, Code 1)
   → Only if first fragment (offset=0) was received

5. An unrecognized extension header is encountered
   → Parameter Problem (Type 4, Code 1 or 2)
   → Depends on option type bits
```

## Validating Incoming ICMPv6 Messages

```python
import struct
import socket
import ipaddress

def validate_icmpv6_message(
    src_addr: str,
    dst_addr: str,
    icmpv6_data: bytes,
    arrived_on_multicast: bool = False,
) -> dict:
    """
    Validate an incoming ICMPv6 message per RFC 4443 rules.
    Returns validation result with reasons for any rejection.
    """
    errors = []
    warnings = []

    if len(icmpv6_data) < 4:
        return {"valid": False, "errors": ["Too short (< 4 bytes)"]}

    icmp_type = icmpv6_data[0]
    icmp_code = icmpv6_data[1]
    checksum = struct.unpack_from("!H", icmpv6_data, 2)[0]
    is_error_message = icmp_type < 128

    # Validate source address
    try:
        src = ipaddress.ip_address(src_addr)
        if src.is_multicast:
            errors.append(f"Error source is multicast: {src_addr}")
        if src.is_unspecified:
            errors.append(f"Error source is unspecified (::): {src_addr}")
        if src.is_loopback and not ipaddress.ip_address(dst_addr).is_loopback:
            warnings.append("Source is loopback but destination is not")
    except ValueError:
        errors.append(f"Invalid source address: {src_addr}")

    # Check anti-loop rule: error in response to error
    # (We check if this IS an error and the invoking packet source is error type)
    if is_error_message and len(icmpv6_data) >= 48:
        # The invoking packet starts at byte 8 of the error message
        invoking_icmpv6_type = icmpv6_data[48] if len(icmpv6_data) > 48 else None
        invoking_next_header = icmpv6_data[14] if len(icmpv6_data) > 14 else None
        if invoking_next_header == 58 and invoking_icmpv6_type and invoking_icmpv6_type < 128:
            errors.append("Error message in response to ICMPv6 error (anti-loop violation)")

    # Check for errors sent to multicast destinations
    try:
        dst = ipaddress.ip_address(dst_addr)
        if dst.is_multicast and is_error_message:
            # Only Type 2 and Type 4 Code 2 are allowed to multicast dest
            if icmp_type not in (2, 4) and not (icmp_type == 4 and icmp_code == 2):
                errors.append(f"Error type {icmp_type} sent to multicast destination (prohibited)")
    except ValueError:
        pass

    return {
        "valid": len(errors) == 0,
        "icmp_type": icmp_type,
        "is_error": is_error_message,
        "errors": errors,
        "warnings": warnings,
    }

# Test validation

tests = [
    ("2001:db8::1", "2001:db8::2", bytes([1, 0, 0, 0, 0, 0, 0, 0]), False),    # Valid
    ("::",         "2001:db8::2", bytes([1, 0, 0, 0, 0, 0, 0, 0]), False),    # Invalid src
    ("2001:db8::1", "ff02::1",   bytes([1, 0, 0, 0, 0, 0, 0, 0]), False),    # Error to multicast
]

for src, dst, data, mcast in tests:
    result = validate_icmpv6_message(src, dst, data, mcast)
    print(f"src={src[:15]}, dst={dst[:15]}: valid={result['valid']}")
    for err in result["errors"]:
        print(f"  ERROR: {err}")
```

## Rate Limiting Rules

```javascript
RFC 4443 Section 2.4: Rate limiting:

"An IPv6 node MUST limit the rate of ICMPv6 error messages it sends."
"The rate-limiting function MUST allow for at least one error message
 per minute."

Linux default: 1 ICMPv6 error per second per destination
Check: /proc/sys/net/ipv6/icmp/ratelimit (in milliseconds)
Default: 1000 (1 second = 1000ms between errors)
```

## Conclusion

ICMPv6 message processing rules exist to prevent error storms and ensure errors are only sent when useful. The anti-loop rule (no error in response to an error) and the multicast restriction (no errors for multicast destinations, except Packet Too Big and Parameter Problem) are the most critical constraints. Rate limiting prevents ICMPv6 from being used as a DoS amplifier. When implementing IPv6 protocol stacks or debugging unusual ICMPv6 behavior, verifying compliance with these rules explains most unexpected behaviors.
