# How to Understand IPv6 Fragment Overlap Prevention

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Fragmentation, Security, RFC 8200, Overlapping Fragments

Description: Understand how IPv6 prevents overlapping fragment attacks, why RFC 8200 mandates discarding overlapping fragments, and how this differs from IPv4 behavior.

## Introduction

IPv4 allowed overlapping fragments, which led to a class of attacks including the Teardrop attack and various intrusion detection evasion techniques. RFC 8200 eliminates this attack surface entirely: if any fragment of an IPv6 packet overlaps with another fragment of the same packet, the entire reassembly buffer must be silently discarded. This rule is simple, unambiguous, and eliminates all overlap-based attacks.

## Why Overlapping Fragments Were Dangerous

```
IPv4 overlapping fragment attacks:

Teardrop attack (1997):
  Fragment 1: offset=0,  length=24 bytes  → bytes 0-23
  Fragment 2: offset=16, length=4 bytes   → bytes 16-19 (overlaps!)
  Result: Malformed packet at reassembly → kernel crash

IDS evasion (Ptacek & Newsham 1998):
  Fragment 1 (seen by IDS): "GET /safe"
  Fragment 2 (overlaps):    overwrite with "GET /attack"
  Result: IDS sees "GET /safe", host sees "GET /attack"

Fragment 0 injection:
  Attacker sends fragment 0 with forged data
  Destination may use first-fragment data without receiving all fragments
```

## IPv6 RFC 8200 Overlap Rule

RFC 8200 Section 4.5 is explicit:

```
RFC 8200 Section 4.5 reassembly rule:

"If any of the fragments being reassembled overlap with any other
fragments being reassembled for the same packet, reassembly of
that packet must be abandoned and all the fragments that have
been received for that packet must be discarded."

Additional rule:
  No ICMPv6 error is sent when fragments are discarded for overlap.
  Overlapping fragments are silently dropped.
  The source will eventually retransmit (TCP) or accept loss (UDP).
```

## Detecting Overlap Attempts

```bash
# Check for fragment overlap indicators in kernel logs
sudo dmesg | grep -i "fragment\|overlap\|frag"

# Watch for reassembly failures (could include overlaps)
watch -n 1 'cat /proc/net/snmp6 | grep -i reasm'
# Ip6ReasmFails increasing rapidly may indicate overlap attacks

# Capture fragments for manual analysis
sudo tcpdump -i eth0 -w /tmp/fragments.pcap "ip6[6] == 44"

# Analyze with tshark for overlapping fragments
tshark -r /tmp/fragments.pcap -Y "ipv6.fragment.overlap == 1"
# Wireshark/tshark detects and flags overlapping fragments
```

## Implementing Overlap Detection

```python
def check_fragment_overlap(fragments: list) -> bool:
    """
    Check if any fragments overlap.
    Each fragment is (offset_bytes, length_bytes).
    Returns True if overlap detected (reassembly must be abandoned).
    """
    # Build a set of all covered byte positions
    covered = []
    for offset, length in fragments:
        covered.append((offset, offset + length - 1))

    # Sort by start position
    covered.sort()

    # Check for overlaps
    for i in range(1, len(covered)):
        prev_end = covered[i-1][1]
        curr_start = covered[i][0]
        if curr_start <= prev_end:
            return True  # Overlap detected

    return False

# Test cases
fragments_ok = [(0, 1448), (1448, 552)]
fragments_overlap = [(0, 1448), (1400, 600)]  # Second starts within first

print(f"Normal fragments overlap: {check_fragment_overlap(fragments_ok)}")
print(f"Overlapping fragments: {check_fragment_overlap(fragments_overlap)}")

def reassemble_or_discard(src: str, dst: str, ident: int,
                          fragments: list) -> bytes | None:
    """
    Attempt reassembly; discard entire buffer if any overlap detected.
    fragments: list of (offset, more_flag, data) tuples
    """
    frag_ranges = [(offset, len(data)) for offset, _, data in fragments]

    if check_fragment_overlap(frag_ranges):
        print(f"OVERLAP DETECTED for ({src}, {dst}, {ident:#x}): discarding all fragments")
        return None  # RFC 8200: silently discard

    # Sort and reassemble
    sorted_frags = sorted(fragments, key=lambda x: x[0])
    result = bytearray()
    for offset, more, data in sorted_frags:
        result[offset:offset + len(data)] = data

    return bytes(result)
```

## Comparison: IPv4 vs IPv6 Overlap Handling

```
IPv4 overlap handling (implementation-defined):
  - RFC 791 does not specify what to do with overlapping fragments
  - Different OS implementations chose different strategies:
    - First-wins: Keep first received data for overlapping range
    - Last-wins: Overwrite with latest received data
    - Result: IDS evasion was possible by targeting the behavior difference

IPv6 overlap handling (RFC 8200, mandatory):
  - Any overlap → silently discard entire reassembly buffer
  - No IDS evasion possible: overlapping = invalid = dropped
  - Consistent across all implementations
  - Attack mitigation at the protocol level, not implementation level
```

## Conclusion

IPv6's overlap prevention rule is a clean security improvement over IPv4. By mandating silent discard of the entire reassembly buffer upon any fragment overlap, RFC 8200 eliminates an entire class of attacks. The rule is easy to implement correctly: compare all fragment ranges before accepting any reassembled data. In practice, legitimate implementations never produce overlapping fragments, so this rule only triggers on malformed or malicious traffic.
