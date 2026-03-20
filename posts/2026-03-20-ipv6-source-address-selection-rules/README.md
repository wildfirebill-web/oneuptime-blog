# How to Apply IPv6 Source Address Selection Rules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Source Address Selection, RFC 6724, Linux, Networking

Description: Deep dive into RFC 6724 source address selection rules with practical examples showing how Linux chooses among multiple IPv6 source addresses for outgoing connections.

## Why Source Address Selection Matters

A host with multiple IPv6 addresses (global, ULA, link-local, temporary) must pick one source address per outgoing connection. The wrong choice causes:
- Packets routed through unexpected paths
- Asymmetric routing and dropped replies
- Privacy leaks (using permanent address instead of temporary)
- Connectivity failures (using deprecated address)

## The 8 Selection Rules in Detail

RFC 6724 evaluates candidate source addresses against the destination using these rules in order:

```
Rule 1: Prefer same address
        If candidate == destination, it wins immediately

Rule 2: Prefer appropriate scope
        Source scope must be >= destination scope
        Link-local source cannot reach global destination

Rule 3: Avoid deprecated addresses
        Preferred lifetime > 0 wins over lifetime = 0

Rule 4: Prefer home address (Mobile IPv6)
        Rarely applicable in fixed networks

Rule 5: Prefer outgoing interface
        Prefer address assigned to the interface used to reach destination

Rule 6: Prefer matching label
        ip addrlabel label(source) == ip addrlabel label(destination)

Rule 7: Prefer temporary address (privacy extensions)
        RFC 4941 temporary address preferred over permanent

Rule 8: Longest matching prefix
        Most bits matching destination prefix wins
```

## Setting Up a Test Environment

```bash
# Add multiple IPv6 addresses to test selection
ip addr add 2001:db8:1::10/64 dev eth0   # Global static
ip addr add fd00::10/64 dev eth0          # ULA
ip addr add fe80::10/64 dev eth0          # Link-local (already present)

# Enable privacy extensions (temporary addresses)
sysctl -w net.ipv6.conf.eth0.use_tempaddr=2

# View all addresses with lifetimes
ip -6 addr show dev eth0
```

## Rule 2: Scope Matching Examples

```bash
# Rule 2: link-local source CANNOT be used for global destination
# ping6 to a global address will NOT use fe80:: source

# Verify: ping to global destination uses global source
ping6 -I eth0 2001:db8::1 &>/dev/null &
ss -6 -n state established sport 0
# Source will be 2001:db8:1::10 or the temporary address — NOT fe80::

# ping to link-local destination uses link-local source
ping6 fe80::1%eth0 &>/dev/null &
ss -6 -n state established sport 0
# Source will be fe80::10
```

## Rule 6: Label Matching

```bash
# Show current policy table labels
ip addrlabel list

# IPv6 global (label 1) prefers global source over ULA (label 13)
# when connecting to a global destination

# Add custom label to prefer ULA source for a specific destination
ip addrlabel add prefix fd00::/7 label 13
ip addrlabel add prefix 2001:db8::/32 label 13  # Same label as ULA

# Now connections to 2001:db8::/32 will prefer fd00:: source
# because labels match
```

## Rule 7: Temporary Address Preference

```bash
# With use_tempaddr=2, temporary addresses are preferred for outgoing

# Check current temporary addresses
ip -6 addr show dev eth0 | grep "scope global temporary"
# inet6 2001:db8:1:0:a1b2:c3d4:e5f6:7890/64 scope global temporary dynamic

# Verify outgoing connections use temporary address
curl -6 https://ifconfig.co/ip
# Returns: 2001:db8:1:0:a1b2:c3d4:... (temporary)

# Force use of specific source address (override selection)
curl -6 --interface 2001:db8:1::10 https://ifconfig.co/ip
# Returns: 2001:db8:1::10 (static)
```

## Rule 8: Longest Prefix Match

```bash
# When connecting to 2001:db8:1::200, which source is preferred?
# Candidate A: 2001:db8:1::10   (same /64 prefix — 64 matching bits)
# Candidate B: 2001:db8:2::10   (different /48 — 48 matching bits)

# Rule 8 picks Candidate A (64 bits match > 48 bits match)

# Python script to visualize prefix matching
python3 << 'EOF'
import ipaddress

destination = ipaddress.IPv6Address('2001:db8:1::200')
candidates = [
    ipaddress.IPv6Address('2001:db8:1::10'),   # same /64
    ipaddress.IPv6Address('2001:db8:2::10'),   # different /48
    ipaddress.IPv6Address('fd00::10'),          # ULA
]

dest_int = int(destination)
for c in candidates:
    c_int = int(c)
    xor = dest_int ^ c_int
    # Count leading zeros = matching prefix bits
    matching = 128 - xor.bit_length() if xor != 0 else 128
    print(f"{c}: {matching} matching bits")
EOF
```

## Rule 3: Deprecated Address Handling

```bash
# Deprecate an address (set preferred lifetime to 0)
ip addr change 2001:db8:1::10/64 dev eth0 preferred_lft 0

# Verify — address shows as "deprecated"
ip -6 addr show dev eth0
# inet6 2001:db8:1::10/64 scope global deprecated

# New connections will NOT use this address (Rule 3)
# Existing connections using it continue to work

# Restore
ip addr change 2001:db8:1::10/64 dev eth0 preferred_lft forever
```

## Debugging Source Address Selection

```bash
#!/bin/bash
# debug-source-selection.sh — Show which source address would be chosen

DESTINATION=${1:-"2001:db8::1"}

echo "Destination: ${DESTINATION}"
echo ""
echo "Available source candidates:"
ip -6 addr show scope global | grep "inet6" | awk '{print $2}'
echo ""

# Open a connection and capture source
python3 << PYEOF
import socket
dest = "${DESTINATION}"
try:
    s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
    s.connect((dest, 80))
    src = s.getsockname()[0]
    print(f"RFC 6724 selected source: {src}")
    s.close()
except Exception as e:
    print(f"Error: {e}")
PYEOF
```

## Application Override

Applications can bypass RFC 6724 by binding explicitly:

```bash
# Python: bind to specific source address
python3 << 'EOF'
import socket

# Create IPv6 socket
s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)

# Bind to specific source (overrides RFC 6724)
s.bind(('2001:db8:1::10', 0))  # explicit source
s.connect(('2001:db8::1', 80))

src = s.getsockname()
print(f"Source: {src[0]}:{src[1]}")
s.close()
EOF
```

## Conclusion

RFC 6724 source address selection follows 8 rules in priority order. The most commonly triggered rules in practice are: Rule 2 (scope — link-local cannot reach global), Rule 5 (prefer interface address), Rule 6 (label matching — ULA stays local), Rule 7 (prefer temporary for privacy), and Rule 8 (longest prefix match). Use `ip addrlabel` to modify label assignments and redirect traffic to specific source prefixes. Debug selection with Python's `socket.connect()` trick — it reveals the OS-chosen source without sending actual traffic.
