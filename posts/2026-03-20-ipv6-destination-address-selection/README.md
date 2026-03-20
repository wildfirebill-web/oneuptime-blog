# How to Control IPv6 Destination Address Selection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Destination Address Selection, RFC 6724, DNS, Dual-Stack, Networking

Description: Understand how RFC 6724 destination address selection sorts DNS results and controls which address a dual-stack host connects to, with configuration and debugging techniques.

## Destination Address Selection Overview

When `getaddrinfo()` returns multiple addresses (e.g., both AAAA and A records), the OS sorts them using RFC 6724 destination address selection. The first address in the sorted list is tried first.

This controls:
- Whether IPv6 or IPv4 is preferred on dual-stack hosts
- Which of multiple IPv6 addresses is tried first
- Whether Teredo or 6to4 addresses are preferred

## The 10 Destination Selection Rules

```text
Rule 1:  Avoid unusable - skip if no valid source exists
Rule 2:  Prefer matching scope - src and dst same scope wins
Rule 3:  Avoid deprecated source - skip if src is deprecated
Rule 4:  Prefer home address - Mobile IPv6 (rarely used)
Rule 5:  Prefer matching label - src label == dst label
Rule 6:  Prefer higher precedence - from policy table
Rule 7:  Prefer native - native IPv6 over 6to4/Teredo
Rule 8:  Prefer smaller scope - link-local before global
Rule 9:  Longest matching prefix - most common bits wins
Rule 10: Keep order unchanged - stable sort, first DNS result stays
```

## Policy Table Precedence (Rule 6)

```bash
# Default policy table precedence values

# Higher = preferred destination

# ::1/128         → precedence 50  (loopback, always first)
# ::/0            → precedence 40  (global IPv6)
# ::ffff:0:0/96   → precedence 35  (IPv4-mapped = IPv4)
# 2002::/16       → precedence 30  (6to4)
# 2001::/32       → precedence  5  (Teredo)
# fc00::/7        → precedence  3  (ULA)

# Result: global IPv6 (40) > IPv4 (35) > 6to4 (30) > Teredo (5)
# IPv6 is preferred over IPv4 on dual-stack hosts by default

ip addrlabel list | sort -k4 -rn
```

## Observing Destination Selection

```bash
# Python: show sorted address list (what getaddrinfo returns)
python3 << 'EOF'
import socket

hostname = "example.com"
results = socket.getaddrinfo(hostname, 80, type=socket.SOCK_STREAM)

print(f"getaddrinfo results for {hostname} (sorted by RFC 6724):")
for i, r in enumerate(results):
    af = "IPv6" if r[0] == socket.AF_INET6 else "IPv4"
    print(f"  {i+1}. [{af}] {r[4][0]}")
EOF

# First result = what most applications connect to
# IPv6 global appears before IPv4 by default
```

## Rule 5: Label Matching in Practice

```bash
# Check labels
ip addrlabel list
# prefix fc00::/7 label 13   ← ULA
# prefix ::/0 label 1         ← default IPv6

# Connecting to a ULA destination (fd00::1):
# - ULA destination has label 13
# - ULA source has label 13  → MATCH (Rule 5 prefers this)
# - Global source has label 1 → NO MATCH

# Connecting to a global destination (2001:db8::1):
# - Global destination has label 1
# - ULA source label 13       → NO MATCH
# - Global source label 1     → MATCH (preferred)

# This ensures ULA stays within the organization
```

## Rule 7: Prefer Native IPv6 Over Tunnels

```bash
# If a host has both a native IPv6 address and a 6to4 address (2002::),
# Rule 7 prefers native IPv6 path to reach native IPv6 destinations

# Demonstrate: add a 6to4 address
# ip addr add 2002:c0a8:010a::1/16 dev eth0  # 6to4 for 192.168.1.10

# When connecting to native 2001:db8::1:
# Native source (2001:db8:1::10) wins over 6to4 source (2002::)
# because Rule 7 marks native transport as preferred

# Check if 6to4 tunnel is active
ip tunnel show | grep sit
ip addr show | grep "2002:"
```

## Rule 9: Longest Prefix Match for Destinations

```bash
# When two IPv6 destinations exist and one shares more prefix bits
# with the selected source, it is preferred

python3 << 'EOF'
import ipaddress

# Source address selected by Rule 5/6/7
source = ipaddress.IPv6Address('2001:db8:1::10')

# Two destinations returned by DNS
destinations = [
    ipaddress.IPv6Address('2001:db8:1::100'),  # same /64
    ipaddress.IPv6Address('2001:db8:2::100'),  # different /48
]

src_int = int(source)
for d in destinations:
    d_int = int(d)
    xor = src_int ^ d_int
    matching = 128 - xor.bit_length() if xor != 0 else 128
    print(f"Destination {d}: {matching} bits match source")
# 2001:db8:1::100 → 64 bits match (preferred, Rule 9)
# 2001:db8:2::100 → 48 bits match
EOF
```

## Influencing Selection with Policy Table

```bash
# Force prefer IPv4 over IPv6 (override Rule 6)
# Add high precedence to IPv4-mapped range

# Prefer IPv4 globally
ip addrlabel add prefix ::ffff:0:0/96 label 4
# Then set precedence by modifying /etc/gai.conf

# /etc/gai.conf - getaddrinfo() policy table for userspace
cat > /etc/gai.conf << 'EOF'
# Uncomment to prefer IPv4 over IPv6
#precedence ::ffff:0:0/96  100
#precedence ::/0             40

# Default: prefer IPv6
label       ::1/128          0
label       ::/0             1
label       ::ffff:0:0/96    4
label       2002::/16        2
label       2001::/32        5
label       fc00::/7         13
precedence  ::1/128         50
precedence  ::/0            40
precedence  ::ffff:0:0/96   35
precedence  2002::/16       30
precedence  2001::/32        5
precedence  fc00::/7         3
EOF
```

## Debugging with getaddrinfo

```bash
# C program to print raw getaddrinfo results
cat > /tmp/gai_debug.c << 'EOF'
#include <stdio.h>
#include <netdb.h>
#include <arpa/inet.h>

int main(int argc, char **argv) {
    if (argc < 2) { puts("Usage: gai_debug <hostname>"); return 1; }
    struct addrinfo hints = {0}, *res, *p;
    hints.ai_socktype = SOCK_STREAM;
    if (getaddrinfo(argv[1], "80", &hints, &res) != 0) return 1;
    int i = 0;
    for (p = res; p; p = p->ai_next, i++) {
        char buf[INET6_ADDRSTRLEN];
        void *addr = p->ai_family == AF_INET6
            ? (void*)&((struct sockaddr_in6*)p->ai_addr)->sin6_addr
            : (void*)&((struct sockaddr_in*)p->ai_addr)->sin_addr;
        inet_ntop(p->ai_family, addr, buf, sizeof(buf));
        printf("%d. [%s] %s\n", i+1,
               p->ai_family == AF_INET6 ? "IPv6" : "IPv4", buf);
    }
    freeaddrinfo(res);
    return 0;
}
EOF
gcc -o /tmp/gai_debug /tmp/gai_debug.c
/tmp/gai_debug example.com
```

## Conclusion

RFC 6724 destination address selection sorts the `getaddrinfo()` result list so applications connect to the best address without any code changes. The default policy prefers global IPv6 (precedence 40) over IPv4 (precedence 35), so dual-stack hosts choose IPv6 automatically. Label matching (Rule 5) keeps ULA addresses local and prevents them from being used for global destinations. Modify `/etc/gai.conf` on Linux to adjust precedence - for example, to temporarily prefer IPv4 while debugging IPv6 issues.
