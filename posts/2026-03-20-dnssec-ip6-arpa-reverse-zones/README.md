# How to Sign ip6.arpa Reverse DNS Zones with DNSSEC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNSSEC, IPv6, Reverse DNS, ip6.arpa, BIND, PTR Records

Description: Sign IPv6 reverse DNS zones (ip6.arpa) with DNSSEC including zone structure, key generation, signing procedures, and DS record delegation for reverse zones.

## IPv6 Reverse DNS Zone Structure

IPv6 reverse DNS uses nibble format in `ip6.arpa`:

```
IPv6 address:  2001:0db8:0001:0002:0003:0004:0005:0006
Reverse zone:  6.0.0.0.5.0.0.0.4.0.0.0.3.0.0.0.2.0.0.0.1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa

Zone for /48:  1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa  (2001:db8:1::/48)
Zone for /32:  8.b.d.0.1.0.0.2.ip6.arpa           (2001:db8::/32)
Zone for /48:  1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa
```

## Creating the ip6.arpa Zone File

```
; /var/named/2001:db8:1::-48.zone (or use a descriptive filename)
; Zone: 1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa (2001:db8:1::/48)

$ORIGIN 1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa.
$TTL 3600

@   IN SOA ns1.example.com. admin.example.com. (
            2026032001  ; serial
            3600        ; refresh
            900         ; retry
            604800      ; expire
            300 )       ; minimum

    IN NS ns1.example.com.
    IN NS ns2.example.com.

; PTR records (nibble-format addresses within the /48)
; 2001:db8:1::1
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR ns1.example.com.

; 2001:db8:1::10 → 0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0
0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0 IN PTR www.example.com.

; 2001:db8:1:1::100
0.0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.0.0 IN PTR host1.example.com.
```

## Script: Generate Reverse Zone from Forward Zone

```python
#!/usr/bin/env python3
# gen-ipv6-reverse.py — Generate ip6.arpa PTR records from AAAA records

import ipaddress
import re
import sys

def ipv6_to_arpa(addr: str) -> str:
    """Convert IPv6 address to nibble-format for ip6.arpa."""
    ip = ipaddress.ip_address(addr)
    # Expand to full 32 hex characters
    full_hex = ip.exploded.replace(":", "")
    # Reverse nibbles and join with dots
    nibbles = list(reversed(full_hex))
    return ".".join(nibbles) + ".ip6.arpa."

def zone_name_for_prefix(prefix: str) -> str:
    """Get the ip6.arpa zone name for a given prefix."""
    net = ipaddress.ip_network(prefix, strict=False)
    prefix_len = net.prefixlen
    nibbles_count = prefix_len // 4  # Each nibble = 4 bits
    full_hex = net.network_address.exploded.replace(":", "")
    relevant_nibbles = full_hex[:nibbles_count]
    return ".".join(reversed(list(relevant_nibbles))) + ".ip6.arpa."

# Parse forward zone and generate reverse records
forward_zone = """
www     IN AAAA 2001:db8:1::10
ns1     IN AAAA 2001:db8:1::1
api     IN AAAA 2001:db8:1::20
"""

prefix = "2001:db8:1::/48"
net = ipaddress.ip_network(prefix, strict=False)

print(f"; Reverse zone for {prefix}")
print(f"; Zone: {zone_name_for_prefix(prefix)}")
print()

for line in forward_zone.strip().splitlines():
    match = re.match(r'(\S+)\s+IN AAAA\s+(\S+)', line)
    if match:
        name, addr = match.groups()
        ip = ipaddress.ip_address(addr)
        if ip in net:
            arpa = ipv6_to_arpa(addr)
            # Remove the zone suffix for relative name
            zone_suffix = zone_name_for_prefix(prefix)
            relative = arpa[:-len(zone_suffix)-1] if arpa.endswith(zone_suffix) else arpa
            print(f"{relative} IN PTR {name}.example.com.")
```

## BIND Configuration for ip6.arpa Zones

```
// /etc/named.conf — Configure ip6.arpa reverse zone

zone "1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa" {
    type master;
    file "/var/named/2001db81-48.reverse.signed";
    key-directory "/var/named/keys";
    auto-dnssec maintain;
    inline-signing yes;
};
```

## Generate Keys and Sign the Reverse Zone

```bash
# Generate keys for the ip6.arpa zone
ZONE="1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa"
KEY_DIR="/var/named/keys"

# ZSK
dnssec-keygen -a ECDSAP256SHA256 -n ZONE "${ZONE}"

# KSK
dnssec-keygen -a ECDSAP256SHA256 -n ZONE -f KSK "${ZONE}"

# Sign the reverse zone
dnssec-signzone \
    -3 $(openssl rand -hex 4) \
    -A -N INCREMENT \
    -o "${ZONE}" \
    -k "${KEY_DIR}/K${ZONE}.+013+KSK_ID" \
    "/var/named/2001db81-48.reverse" \
    "${KEY_DIR}/K${ZONE}.+013+ZSK_ID"

# Verify PTR records have signatures
grep -A2 "PTR" 2001db81-48.reverse.signed | grep RRSIG | head -3

# Test lookup
dig +dnssec PTR \
    0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa \
    @localhost
```

## Delegating ip6.arpa to Your Authoritative Server

```
# Contact your RIR (ARIN, RIPE, APNIC) or ISP to delegate
# They configure NS and DS records for your ip6.arpa zone

# Submit DS record to RIR via their portal
# Extract DS record from KSK:
dnssec-dsfromkey K${ZONE}.+013+KSK_ID.key

# Output example:
# 1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa IN DS 12345 13 2 <hash>

# After delegation, verify NS records are published:
dig NS 1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa +short

# Verify DS record is published at parent:
dig DS 1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa @a.ip6-servers.arpa
```

## Verify Reverse DNSSEC

```bash
# Test full DNSSEC chain for PTR record
dig +dnssec PTR \
    0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa

# Check for 'ad' flag (Authenticated Data)
# And RRSIG record alongside PTR

# Reverse lookup with validation
host -t PTR 2001:db8:1::10

# DNSViz visualization (shows full DNSSEC chain)
# https://dnsviz.net/ — enter the ip6.arpa zone name
```

## Conclusion

DNSSEC signing of ip6.arpa reverse zones follows the same process as forward zones. The key consideration is the zone name: convert your IPv6 prefix to nibble-format ip6.arpa notation (nibbles reversed, dotted). Generate separate ZSK and KSK per zone, sign with `dnssec-signzone -3` (NSEC3), and configure BIND with `auto-dnssec maintain; inline-signing yes` for automatic re-signing. Submit the KSK's DS record to your RIR or ISP to complete the DNSSEC chain. Verify with `dig +dnssec PTR` and check for the `ad` flag in responses from validating resolvers.
