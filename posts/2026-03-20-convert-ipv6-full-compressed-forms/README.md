# How to Convert IPv6 Addresses Between Full and Compressed Forms

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Networking, Address Conversion, Python, Go, JavaScript

Description: Convert IPv6 addresses between full expanded form and compressed form in Python, Go, and JavaScript, following RFC 5952 canonical notation rules.

## Introduction

IPv6 addresses can be written in full (32 hex digits with colons) or compressed form (leading zeros dropped, `::` for zero runs). RFC 5952 defines the canonical compressed form used in configuration files, DNS records, and logs.

## Python

```python
import ipaddress

def expand(ipv6: str) -> str:
    """Return full expanded IPv6 address."""
    return ipaddress.IPv6Address(ipv6).exploded

def compress(ipv6: str) -> str:
    """Return canonical compressed IPv6 address (RFC 5952)."""
    return ipaddress.IPv6Address(ipv6).compressed

examples = [
    "2001:0db8:0000:0000:0000:0000:0000:0001",
    "2001:db8::1",
    "fe80::1",
    "0:0:0:0:0:0:0:1",
    "::",
]

for addr in examples:
    print(f"Input:    {addr}")
    print(f"Expanded: {expand(addr)}")
    print(f"Compressed: {compress(addr)}")
    print()
```

## Go

```go
package main

import (
    "fmt"
    "net"
)

func expand(ipv6 string) string {
    ip := net.ParseIP(ipv6)
    if ip == nil {
        return ""
    }
    // Format as full 32-hex-digit string with colons
    b := ip.To16()
    return fmt.Sprintf("%04x:%04x:%04x:%04x:%04x:%04x:%04x:%04x",
        uint(b[0])<<8|uint(b[1]),
        uint(b[2])<<8|uint(b[3]),
        uint(b[4])<<8|uint(b[5]),
        uint(b[6])<<8|uint(b[7]),
        uint(b[8])<<8|uint(b[9]),
        uint(b[10])<<8|uint(b[11]),
        uint(b[12])<<8|uint(b[13]),
        uint(b[14])<<8|uint(b[15]),
    )
}

func compress(ipv6 string) string {
    ip := net.ParseIP(ipv6)
    if ip == nil {
        return ""
    }
    return ip.String() // Go's net package outputs compressed form
}

func main() {
    addrs := []string{
        "2001:0db8:0000:0000:0000:0000:0000:0001",
        "fe80::1",
        "::1",
        "::",
    }
    for _, a := range addrs {
        fmt.Printf("Input:      %s\n", a)
        fmt.Printf("Expanded:   %s\n", expand(a))
        fmt.Printf("Compressed: %s\n\n", compress(a))
    }
}
```

## JavaScript

```javascript
function expandIPv6(ipv6) {
    // Expand :: notation
    let [left, right] = ipv6.includes('::')
        ? ipv6.split('::')
        : [ipv6, null];

    const leftGroups  = left ? left.split(':') : [];
    const rightGroups = right ? right.split(':') : [];
    const missing     = 8 - leftGroups.length - rightGroups.length;
    const middle      = Array(missing).fill('0');

    const all = [...leftGroups, ...middle, ...rightGroups]
        .map(g => g.padStart(4, '0'));

    return all.join(':');
}

function compressIPv6(full) {
    // Drop leading zeros per group
    let groups = full.split(':').map(g => parseInt(g, 16).toString(16));

    // Find longest run of 0 groups
    let bestStart = -1, bestLen = 0, cur = -1, len = 0;
    groups.forEach((g, i) => {
        if (g === '0') { if (cur < 0) cur = i; len++; }
        else { if (len > bestLen) { bestStart = cur; bestLen = len; } cur = -1; len = 0; }
    });
    if (len > bestLen) { bestStart = cur; bestLen = len; }

    if (bestLen > 1) {
        groups.splice(bestStart, bestLen, '');
        if (bestStart === 0)  groups.unshift('');
        if (bestStart + bestLen === 8) groups.push('');
    }
    return groups.join(':');
}

console.log(expandIPv6("2001:db8::1"));     // 2001:0db8:0000:0000:0000:0000:0000:0001
console.log(compressIPv6("2001:0db8:0000:0000:0000:0000:0000:0001")); // 2001:db8::1
```

## RFC 5952 Canonical Rules Summary

```
1. Remove all leading zeros from each group
2. Replace the LONGEST consecutive all-zero group run with ::
3. If two runs are equal length, replace the FIRST one
4. :: represents exactly one contiguous run of zeros
5. Lowercase hexadecimal letters
```

## Conclusion

Python's `ipaddress.IPv6Address` handles both expansion and compression natively with `.exploded` and `.compressed` properties following RFC 5952. Go's `net.ParseIP().String()` outputs compressed form. In JavaScript, manual implementation requires handling `::` expansion and finding the longest zero run for compression.
