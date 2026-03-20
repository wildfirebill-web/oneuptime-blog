# How to Handle IPv6 Fragment Reassembly

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Fragmentation, Reassembly, RFC 8200, Networking

Description: Understand how IPv6 fragment reassembly works at the destination, the reassembly timer, handling out-of-order fragments, and common reassembly failure scenarios.

## Introduction

IPv6 fragment reassembly is performed exclusively at the destination. When a source fragments a packet, it assigns all fragments the same Identification value. The destination collects all fragments with the same (source address, destination address, Identification) key and reconstructs the original packet. RFC 8200 defines the reassembly process and its constraints.

## Reassembly Algorithm

```
IPv6 Reassembly Steps (RFC 8200):

1. Receive fragment with Fragment Header (NH=44)
2. Extract reassembly key: (Source IP, Dest IP, Identification)
3. Create or find reassembly buffer for this key
4. Place fragment data at its Fragment Offset position
5. Mark the range [offset, offset + length) as received
6. Check completion conditions:
   a. Fragment with M=0 received (last fragment known)
   b. All byte ranges from 0 to last-fragment-end are filled
7. If complete: reconstruct original packet, deliver to upper layer
8. If not complete within timeout: discard buffer, send ICMPv6 Time Exceeded
```

## Reassembly Timer and Timeout

```bash
# Linux reassembly timeout: check kernel parameters
cat /proc/sys/net/ipv6/netfilter/ip6_frag_time
# Default: 60 seconds (in jiffies, system-dependent)

# Alternative location for raw timeout
cat /proc/sys/net/netfilter/nf_conntrack_frag6_timeout
# Default: 60000000 (60 seconds in nanoseconds)

# Check reassembly memory limits
cat /proc/sys/net/ipv6/netfilter/ip6_frag_high_thresh
# Maximum memory used for reassembly (bytes)

cat /proc/sys/net/ipv6/netfilter/ip6_frag_low_thresh
# Memory at which fragment queues start being dropped

# Monitor reassembly failures
cat /proc/net/snmp6 | grep -i reasm
# Ip6ReasmReqds   - total reassembly requests
# Ip6ReasmOKs     - successful reassemblies
# Ip6ReasmFails   - failed reassemblies (timeout or error)
```

## Reassembly Failure: ICMPv6 Time Exceeded

When the reassembly timer expires, the destination sends an ICMPv6 Time Exceeded message (Type 3, Code 1) back to the source of the first fragment received:

```bash
# Capture ICMPv6 reassembly timeout messages
sudo tcpdump -i eth0 -n "icmp6 and ip6[40] == 3 and ip6[41] == 1"
# Type 3 = Time Exceeded, Code 1 = Fragment Reassembly Time Exceeded

# If you see these, check:
# 1. Are all fragments arriving? (network drops)
# 2. Is the reassembly buffer large enough?
# 3. Is the source sending within the timeout window?
```

## Implementing Fragment Reassembly in Python

```python
import time
from collections import defaultdict

class IPv6Reassembler:
    """Simple IPv6 fragment reassembler per RFC 8200."""

    TIMEOUT = 60  # seconds

    def __init__(self):
        # Key: (src, dst, identification) -> reassembly state
        self._buffers = {}

    def _key(self, src: str, dst: str, identification: int) -> tuple:
        return (src, dst, identification)

    def add_fragment(self, src: str, dst: str, identification: int,
                     offset_bytes: int, more_fragments: bool, data: bytes) -> bytes | None:
        """
        Add a fragment to the reassembly buffer.
        Returns the reassembled packet bytes if complete, else None.
        """
        key = self._key(src, dst, identification)

        if key not in self._buffers:
            self._buffers[key] = {
                "received": {},   # offset -> data
                "total_length": None,
                "created": time.time(),
            }

        buf = self._buffers[key]

        # Check timeout
        if time.time() - buf["created"] > self.TIMEOUT:
            del self._buffers[key]
            return None  # Would send ICMPv6 Time Exceeded here

        # Store this fragment
        buf["received"][offset_bytes] = data

        # If this is the last fragment, we know the total length
        if not more_fragments:
            buf["total_length"] = offset_bytes + len(data)

        # Check if reassembly is complete
        if buf["total_length"] is not None:
            # Verify all bytes from 0 to total_length are covered
            covered = set()
            for frag_offset, frag_data in buf["received"].items():
                for b in range(frag_offset, frag_offset + len(frag_data)):
                    covered.add(b)

            if len(covered) == buf["total_length"]:
                # Reassemble in order
                result = bytearray(buf["total_length"])
                for frag_offset, frag_data in buf["received"].items():
                    result[frag_offset:frag_offset + len(frag_data)] = frag_data
                del self._buffers[key]
                return bytes(result)

        return None  # Not complete yet

# Example reassembly
reassembler = IPv6Reassembler()
src, dst, ident = "2001:db8::1", "2001:db8::2", 0xABCD1234

# Receive fragments out of order
result = reassembler.add_fragment(src, dst, ident, 1448, False, b"B" * 552)
print(f"After fragment 2: {result}")  # None - incomplete

result = reassembler.add_fragment(src, dst, ident, 0, True, b"A" * 1448)
print(f"After fragment 1: {len(result)} bytes reassembled" if result else "None")
```

## Reassembly Security Considerations

Fragment reassembly has historically been an attack vector:

```
Known fragment reassembly attacks:

1. Fragment flooding: Sending many incomplete fragment sets
   → Exhausts reassembly buffer memory
   → Mitigated by ip6_frag_high_thresh limit

2. Overlapping fragments (Teardrop attack):
   → RFC 8200: Destination MUST silently discard on overlap
   → No overlapping fragments are permitted in IPv6

3. Atomic fragment attacks (RFC 6946):
   → Spoofed ICMPv6 PTB with large MTU forces atomic fragments
   → Attacker can predict Identification and inject data
   → Mitigated by using random Identification values (RFC 7739)
```

## Conclusion

IPv6 fragment reassembly is simpler than IPv4 in some respects (no overlapping fragments allowed) but places the full burden on the destination. The 60-second reassembly timer means all fragments of a packet must arrive within one minute. Reassembly buffers are memory-limited, so high fragment rates can exhaust them. In practice, IPv6 fragmentation should be avoided through proper PMTUD implementation — fragmentation is a fallback mechanism, not a primary design pattern.
