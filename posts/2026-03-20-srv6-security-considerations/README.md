# How to Understand SRv6 Security Considerations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SRv6, Security, RFC 8754, HMAC, ACL, Networking

Description: Understand SRv6 security threats and mitigations including SRH spoofing, unauthorized SID invocation, HMAC authentication, and perimeter filtering.

## Introduction

SRv6's source routing capability introduces security concerns: a malicious source can craft SRH packets to invoke internal SIDs, potentially bypassing firewall rules or accessing private services. RFC 8754 and operational best practices address these threats.

## Key Security Threats

### 1. Unauthorized SID Invocation

An external host crafts a packet with a crafted SRH pointing to internal End.DX4 or End.DT6 SIDs:

```text
Attack:
  External packet:
    dst=5f00:internal:1:0:e000::  (End.DT6 for internal VPN)
    SRH: [internal_server_sid]

Impact: Traffic bypasses perimeter security controls
```

### 2. SRH Amplification

A packet with a large SRH (many SIDs) consumes more processing than a plain IPv6 packet.

### 3. Topology Disclosure

SRH segment lists expose network topology (SID addresses = node addresses).

## Mitigation 1: Perimeter Filtering (Most Important)

```bash
# Block SRv6 packets from external sources at the network boundary

# Only allow SRH packets from trusted internal sources

# ip6tables on edge router
# Drop any packet with Routing Header from external sources
ip6tables -A FORWARD \
  -i eth0 \                          # External interface
  -m ipv6header --soft --header routing \
  -j DROP

# Or use a more specific match for RH Type 4 (SRH)
ip6tables -A INPUT \
  -m ipv6header --soft --header routing \
  -m rt --rt-type 4 \
  -j DROP

# Cisco: ACL to block SRH from external
# ipv6 access-list BLOCK_SRH_EXTERNAL
#   deny ipv6 any any routing-type 4
#   permit ipv6 any any
```

## Mitigation 2: HMAC Authentication (SRH Integrity)

RFC 8754 defines an HMAC TLV for the SRH that authenticates the segment list.

```bash
# Linux: configure HMAC key
ip sr hmac set 1 SHA256 \
  aabbccddeeff00112233445566778899aabbccddeeff00112233445566778899

# Require HMAC on incoming SRv6 packets
sysctl net.ipv6.conf.eth0.seg6_require_hmac=1

# Packets without valid HMAC are dropped
# Legitimate sources must include the correct HMAC TLV
```

```python
from scapy.all import *

def compute_srh_hmac(segment_list: list, key: bytes, key_id: int) -> bytes:
    """
    Compute HMAC-SHA256 over the SRH segment list.
    RFC 8754 §4
    """
    import hmac
    import hashlib
    import struct

    # Build the message: key_id (4 bytes) || segment_list
    message = struct.pack("!I", key_id)
    for sid in segment_list:
        # Each SID is 16 bytes (128 bits)
        message += socket.inet_pton(socket.AF_INET6, sid)

    mac = hmac.new(key, message, hashlib.sha256).digest()
    return mac[:16]  # Truncate to 128 bits
```

## Mitigation 3: SID Access Control Lists

Restrict which sources can invoke each SID.

```bash
# Only allow packets from the SRv6 controller to invoke TE SIDs
# On the router owning 5f00:1:1::/48

ip6tables -A INPUT \
  -d 5f00:1:1:0:e001:: \             # End.X SID
  ! -s 5f00:controller::/32 \        # Must come from controller subnet
  -j DROP

ip6tables -A INPUT \
  -d 5f00:1:1:0:e001:: \
  -s 5f00:controller::/32 \
  -j ACCEPT
```

## Mitigation 4: Infrastructure ACL (iACL)

```text
! Cisco IOS-XR infrastructure ACL
ipv6 access-list INFRA_ACL
  remark Block SRH to internal infrastructure SIDs from external
  deny ipv6 any 5f00::/16 routing-type 4 any
  permit ipv6 any any
!
interface GigabitEthernet0/0/0/0
 description EXTERNAL FACING
 ipv6 access-group INFRA_ACL ingress
```

## Mitigation 5: Topology Hiding

```bash
# Use 5f00::/16 or your own allocated SRv6 space
# (do not use public addresses for internal SIDs)

# Summarize SRv6 locators at the border
# Advertise only an aggregate, not individual /48 locators
ip -6 route add 5f00:1::/32 dev null  # Aggregate black hole at border
# Only /128 routes to actual SIDs installed on the node

# For BGP: apply a route policy to filter SRv6 SIDs from external BGP
# Only accept SID advertisements from trusted internal ASes
```

## Security Checklist

```yaml
srv6_security_checklist:
  perimeter:
    - [ ] Block SRH (RH Type 4) from external sources at all edges
    - [ ] Block SRv6 SID prefixes in ingress ACLs

  authentication:
    - [ ] Enable HMAC on all SRv6-capable interfaces
    - [ ] Rotate HMAC keys regularly

  access_control:
    - [ ] ACLs on End.X and End.DT SIDs restrict to authorized sources
    - [ ] Controller-to-PE authentication (BGP MD5 or TCP-AO)

  monitoring:
    - [ ] Alert on unexpected SRH packets at edge
    - [ ] Log HMAC validation failures
    - [ ] Monitor for SRH amplification attacks (high SRH packet rate)
```

## Conclusion

SRv6 security requires treating SIDs as network resources that need access control, just like MPLS labels. Perimeter filtering is the most critical control. HMAC provides cryptographic SRH integrity. Use OneUptime to monitor for anomalous traffic patterns that may indicate SRv6 abuse attempts.
