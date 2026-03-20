# How to Understand IPv6 Address Selection (RFC 6724)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RFC 6724, Address Selection, Networking, Linux

Description: Understand how RFC 6724 governs IPv6 source and destination address selection, including the default policy table, selection rules, and how hosts choose among multiple addresses.

## What Is RFC 6724?

RFC 6724 defines the default behavior for selecting source and destination IPv6 addresses when a host has multiple addresses available. Every IPv6 host follows these rules when initiating a connection.

Two selection algorithms work together:
- **Source address selection**: which local address to use as the packet source
- **Destination address selection**: which remote address to contact when DNS returns multiple results

## Address Scopes

RFC 6724 classifies addresses by scope, from narrowest to broadest:

| Scope | Value | Example |
|---|---|---|
| Interface-local | 1 | ::1 (loopback) |
| Link-local | 2 | fe80::/10 |
| Site-local (deprecated) | 5 | fec0::/10 |
| Global | 14 | 2001:db8::/32 |

Higher scope is preferred for global communication.

## The Default Policy Table

RFC 6724 defines a default policy table with label and precedence values:

```
Prefix             Precedence  Label
::1/128            50          0      # Loopback
::/0               40          1      # Default (IPv6)
::ffff:0:0/96      35          4      # IPv4-mapped
2002::/16          30          2      # 6to4
2001::/32          5           5      # Teredo
fc00::/7           3           13     # ULA
::/96              1           3      # IPv4-compatible (obsolete)
fec0::/10          1           11     # Site-local (obsolete)
3ffe::/16          1           12     # 6bone (obsolete)
```

Higher precedence = preferred destination. Labels are used to match source and destination — same label is preferred.

## Viewing the Policy Table on Linux

```bash
# Show the current address selection policy table
ip addrlabel list

# Default output shows prefix/len label pairs
# Example:
#   prefix ::1/128 label 0
#   prefix ::/0 label 1
#   prefix 2002::/16 label 2
#   prefix ::/96 label 3
#   prefix ::ffff:0:0/96 label 4
#   prefix 2001::/32 label 5
#   prefix fc00::/7 label 13
#   prefix fec0::/10 label 11
#   prefix 3ffe::/16 label 12
```

Precedence is stored in `/proc/sys/net/ipv6/conf/all/use_tempaddr` for privacy extensions, but the policy table itself is managed via `ip addrlabel`.

## Source Address Selection: 8 Rules

RFC 6724 source address selection evaluates candidates using 8 rules in order. The first rule that produces a winner stops evaluation:

```
Rule 1: Prefer same address (if source == destination, done)
Rule 2: Prefer appropriate scope (source scope >= destination scope)
Rule 3: Avoid deprecated addresses
Rule 4: Prefer home addresses (Mobile IPv6)
Rule 5: Prefer outgoing interface address
Rule 6: Prefer matching label (source label == destination label)
Rule 7: Prefer temporary addresses (privacy extensions)
Rule 8: Use longest matching prefix
```

## Destination Address Selection: 10 Rules

When DNS returns multiple addresses, the destination list is sorted:

```
Rule 1: Avoid unusable destinations
Rule 2: Prefer matching scope
Rule 3: Avoid deprecated source
Rule 4: Prefer home address
Rule 5: Prefer matching label
Rule 6: Prefer higher precedence
Rule 7: Prefer native transport (over encapsulated/tunneled)
Rule 8: Prefer smaller scope
Rule 9: Use longest matching prefix
Rule 10: Leave order unchanged (stable sort)
```

## Practical Example: Dual-Stack Host

```bash
# A dual-stack host has these addresses:
ip addr show eth0
# 2: eth0
#    inet  192.168.1.10/24
#    inet6 2001:db8::10/64
#    inet6 fe80::1/64

# DNS returns both A and AAAA records:
# host example.com
# example.com has address 93.184.216.34
# example.com has IPv6 address 2606:2800:220:1:248:1893:25c8:1946

# RFC 6724 rule 6: IPv6 destination (label 1) matches IPv6 source (label 1)
# IPv4 destination (label 4, IPv4-mapped) does NOT match IPv6 source (label 1)
# Result: IPv6 path is preferred

# Confirm with getaddrinfo trace:
python3 -c "
import socket
results = socket.getaddrinfo('example.com', 80, type=socket.SOCK_STREAM)
for r in results:
    print(r[4])
"
# [('2606:2800:...', 80, 0, 0), ('93.184.216.34', 80)]
# IPv6 listed first = preferred
```

## ULA vs Global Address Selection

```bash
# When a host has both ULA (fc00::/7) and global addresses,
# destination scope drives source selection

# ULA destination → ULA source preferred (label 13 matches label 13)
# Global destination → Global source preferred (label 1 matches label 1)

# Test:
# Connect to ULA destination
strace -e trace=connect curl -6 http://[fd00::1]/ 2>&1 | grep "sin6_addr"

# Verify source address chosen
ss -6 -n state established | head -5
```

## Privacy Extensions and Rule 7

RFC 4941 privacy extensions generate temporary addresses with random interface IDs. RFC 6724 Rule 7 prefers temporary addresses for outgoing connections to enhance privacy:

```bash
# Check privacy extension settings
sysctl net.ipv6.conf.eth0.use_tempaddr
# 0 = disabled
# 1 = generate but prefer public
# 2 = generate and prefer temporary (default on many systems)

# Verify which address is chosen for outgoing connections
curl -6 https://ifconfig.co
# Should return your temporary address if use_tempaddr=2
```

## Conclusion

RFC 6724 address selection is automatic but configurable. The policy table assigns labels and precedence to prefixes — same-label source/destination pairs are preferred, and higher precedence destinations are tried first. The 8-rule source selection and 10-rule destination sorting algorithms work together to choose the best path. Key practical outcomes: IPv6 is preferred over IPv4 on dual-stack hosts (labels differ), temporary addresses are preferred for privacy (Rule 7), and ULA addresses stay local (scope matching). Modify the policy table with `ip addrlabel` to override defaults.
