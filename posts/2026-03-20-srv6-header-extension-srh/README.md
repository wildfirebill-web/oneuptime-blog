# How to Understand SRv6 Header Extension (SRH)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, SRH, Segment Routing Header, IPv6, RFC 8754, Networking

Description: Understand the structure of the IPv6 Segment Routing Header (SRH), how it encodes the segment list, and how routers process it hop by hop.

## Introduction

The Segment Routing Header (SRH) is an IPv6 Routing Header (Type 4), defined in RFC 8754. It carries the ordered list of Segment Identifiers (SIDs) that define the explicit path or service chain a packet must traverse.

## SRH Wire Format

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Next Header  |  Hdr Ext Len  | Routing Type=4| Segments Left |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Last Entry   |     Flags     |              Tag              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|            Segment List[0] (128 bits, destination)            |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|            Segment List[1]                                    |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
...
|                                                               |
|            Segment List[N] (first SID to visit)               |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Optional TLV objects                                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## Field Descriptions

| Field | Bits | Description |
|---|---|---|
| Next Header | 8 | Protocol of the next header after SRH |
| Hdr Ext Len | 8 | Length in 8-octet units, not counting first 8 bytes |
| Routing Type | 8 | Must be 4 (SRH) |
| Segments Left (SL) | 8 | Number of SIDs still to visit |
| Last Entry | 8 | Index of the last SID in Segment List |
| Flags | 8 | Reserved; must be 0 |
| Tag | 16 | Groups related packets (zero if unused) |
| Segment List[n] | 128×n | Ordered SID list, Segment List[0] = final dest |

## Segment List Ordering

**Critical**: The segment list is stored in **reverse order**. The first SID to visit is at index [Last Entry], and the final destination is at index [0].

```text
Example path: A → B → C → D

Segment List in SRH:
  [0] = D (final destination SID)
  [1] = C
  [2] = B
  Segments Left = 2 (pointing to B initially)

IPv6 Destination Address = Segment List[SL] = Segment List[2] = B
```

## SRH Processing Algorithm

```python
def process_srh(packet):
    """
    RFC 8754 SRH processing at an SRv6 endpoint.
    Called when the packet's destination matches a local SID.
    """
    srh = packet.srh
    sid_function = lookup_local_sid(packet.ipv6.destination)

    if sid_function is None:
        # Not a local SID - transit processing
        forward_packet(packet)
        return

    # Execute the SID function
    execute_function(sid_function, packet)

    # Update SRH: advance to next segment
    if srh.segments_left > 0:
        srh.segments_left -= 1
        # New destination = next segment in list
        next_sid = srh.segment_list[srh.segments_left]
        packet.ipv6.destination = next_sid
        forward_packet(packet)
    else:
        # segments_left == 0: deliver to final destination
        deliver_to_upper_layer(packet)
```

## Parsing SRH with Scapy

```python
from scapy.all import *
from scapy.contrib.segment_routing import IPv6ExtHdrSegmentRouting

# Craft a packet with SRH (2 waypoints + final destination)

pkt = (
    IPv6(src="2001:db8::1", dst="5f00:1:2:0:e001::") /
    IPv6ExtHdrSegmentRouting(
        type=4,
        segleft=2,
        lastentry=2,
        addresses=[
            "5f00:3:1:0:e000::",  # [0] final destination
            "5f00:2:1:0:e001::",  # [1] second waypoint
            "5f00:1:2:0:e001::",  # [2] first waypoint (== dst)
        ]
    ) /
    IPv6(src="2001:db8::1", dst="2001:db8::100") /  # original packet
    TCP(dport=80)
)

pkt.show2()

# Parse from pcap
pkts = rdpcap("capture.pcap")
for p in pkts:
    if IPv6ExtHdrSegmentRouting in p:
        srh = p[IPv6ExtHdrSegmentRouting]
        print(f"SL={srh.segleft}, segments={srh.addresses}")
```

## SRH TLV Objects

Optional TLV (Type-Length-Value) objects can be appended after the segment list:

- **Pad1 (type 0)**: Single-byte alignment padding
- **PadN (type 1)**: Multi-byte alignment padding
- **HMAC TLV (type 5)**: Authentication of SRH (RFC 8754 §8)
- **NSH Carrier TLV**: For service function chaining integration

## Conclusion

The SRH is the carrier of SRv6's explicit routing information. Its reverse-ordered segment list and Segments Left pointer enable deterministic hop-by-hop processing. Understanding SRH parsing is essential for Wireshark analysis and custom SRv6 tooling. Use OneUptime to monitor end-to-end paths that traverse SRv6 segments.
