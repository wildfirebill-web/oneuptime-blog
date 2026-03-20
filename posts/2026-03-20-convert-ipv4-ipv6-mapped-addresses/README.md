# How to Convert Between IPv4 and IPv6-Mapped Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, IPv6, Address Mapping, Python, Go, Networking

Description: Convert between IPv4 addresses and their IPv6-mapped equivalents (::ffff:0:0/96) in Python, Go, and JavaScript to support dual-stack applications.

## Introduction

IPv6-mapped IPv4 addresses have the form `::ffff:<IPv4>` (e.g., `::ffff:192.168.1.1`). They allow IPv6-only sockets to represent IPv4 connections, and are critical in dual-stack networking. Many operating systems present incoming IPv4 connections on IPv6 sockets using this format.

## Format

```
IPv4:          192.168.1.100
IPv6-mapped:   ::ffff:192.168.1.100
Full notation: 0000:0000:0000:0000:0000:ffff:c0a8:0164
```

## Python

```python
import ipaddress

def ipv4_to_mapped(ipv4_str: str) -> str:
    v4 = ipaddress.IPv4Address(ipv4_str)
    mapped = ipaddress.IPv6Address(f"::ffff:{ipv4_str}")
    return str(mapped)

def mapped_to_ipv4(ipv6_str: str) -> str | None:
    v6 = ipaddress.IPv6Address(ipv6_str)
    if v6.ipv4_mapped:
        return str(v6.ipv4_mapped)
    return None

# Examples
print(ipv4_to_mapped("192.168.1.100"))           # ::ffff:192.168.1.100
print(ipv4_to_mapped("10.0.0.1"))                # ::ffff:a00:1
print(mapped_to_ipv4("::ffff:192.168.1.100"))    # 192.168.1.100
print(mapped_to_ipv4("2001:db8::1"))             # None (not mapped)
```

## Go

```go
package main

import (
    "fmt"
    "net"
)

func ipv4ToMapped(ipv4 string) string {
    ip := net.ParseIP(ipv4).To4()
    if ip == nil {
        return ""
    }
    // Construct ::ffff:x.x.x.x
    mapped := net.IP{0,0,0,0, 0,0,0,0, 0,0,0xff,0xff,
                     ip[0], ip[1], ip[2], ip[3]}
    return mapped.String()
}

func mappedToIPv4(ipv6 string) string {
    ip := net.ParseIP(ipv6)
    if ip == nil {
        return ""
    }
    v4 := ip.To4()
    if v4 == nil {
        return ""
    }
    return v4.String()
}

func main() {
    fmt.Println(ipv4ToMapped("192.168.1.100"))         // ::ffff:192.168.1.100
    fmt.Println(mappedToIPv4("::ffff:192.168.1.100"))  // 192.168.1.100
    fmt.Println(mappedToIPv4("2001:db8::1"))           // "" (not IPv4-mapped)
}
```

## JavaScript

```javascript
function ipv4ToMapped(ipv4) {
    const parts = ipv4.split('.').map(Number);
    const hex = parts.map(p => p.toString(16).padStart(2, '0'));
    const group1 = hex[0] + hex[1];
    const group2 = hex[2] + hex[3];
    return `::ffff:${parseInt(group1, 16)}.${parseInt(group2.slice(0,2),16)}.` +
           // simpler form:
           `${ipv4}`.replace(/^/, '');
}

// Cleaner version using string template
function toMapped(ipv4) { return `::ffff:${ipv4}`; }

function fromMapped(ipv6) {
    const m = ipv6.match(/^::ffff:(\d+\.\d+\.\d+\.\d+)$/i);
    return m ? m[1] : null;
}

console.log(toMapped("192.168.1.100"));           // ::ffff:192.168.1.100
console.log(fromMapped("::ffff:192.168.1.100"));  // 192.168.1.100
```

## Detecting Mapped Addresses in Dual-Stack Servers

```python
import socket, ipaddress

def normalize_client_ip(addr: str) -> str:
    """Strip IPv6-mapped prefix to return the real IPv4 address."""
    try:
        v6 = ipaddress.IPv6Address(addr)
        return str(v6.ipv4_mapped) if v6.ipv4_mapped else addr
    except ValueError:
        return addr  # already IPv4

# When accept() returns "::ffff:10.0.0.5"
print(normalize_client_ip("::ffff:10.0.0.5"))  # 10.0.0.5
```

## Conclusion

IPv6-mapped IPv4 addresses follow the `::ffff:0:0/96` prefix. Python's `ipaddress` module exposes `ipv4_mapped` directly; Go's `net.IP.To4()` handles the conversion; in JavaScript simple string manipulation works for the common `::ffff:x.x.x.x` form. Always normalize mapped addresses when logging or performing access control checks.
