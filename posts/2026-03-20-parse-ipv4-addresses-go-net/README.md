# How to Parse IPv4 Addresses Using Go net.ParseIP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, IPv4, net.ParseIP, Networking, IP Addresses, Validation

Description: Learn how to parse, validate, and manipulate IPv4 addresses in Go using the net package's ParseIP, IP, and related types.

## Parsing IPv4 Addresses

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    // net.ParseIP parses both IPv4 and IPv6 addresses
    ip := net.ParseIP("192.168.1.100")
    if ip == nil {
        fmt.Println("Invalid IP address")
        return
    }

    // To4() returns a 4-byte representation if IPv4, nil otherwise
    ipv4 := ip.To4()
    if ipv4 != nil {
        fmt.Printf("IPv4 address: %s\n", ipv4)
        // Access individual octets
        fmt.Printf("Octets: %d.%d.%d.%d\n", ipv4[0], ipv4[1], ipv4[2], ipv4[3])
    } else {
        fmt.Println("Not an IPv4 address")
    }
}
```

## Validating IPv4 Addresses

```go
package main

import (
    "fmt"
    "net"
)

// IsValidIPv4 returns true if the string is a valid IPv4 address
func IsValidIPv4(s string) bool {
    ip := net.ParseIP(s)
    if ip == nil {
        return false
    }
    // To4() returns non-nil only for IPv4 addresses
    return ip.To4() != nil
}

func main() {
    testCases := []string{
        "192.168.1.1",     // Valid
        "10.0.0.256",      // Invalid (octet > 255)
        "172.16.0.1",      // Valid private
        "::1",             // IPv6 loopback (not IPv4)
        "8.8.8.8",         // Valid public
        "not-an-ip",       // Invalid
        "1.2.3",           // Invalid (too few octets)
    }

    for _, tc := range testCases {
        valid := IsValidIPv4(tc)
        fmt.Printf("%-20s -> valid: %v\n", tc, valid)
    }
}
```

## Converting Between IPv4 String and net.IP

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    // String -> net.IP
    ipStr := "10.0.0.1"
    ip := net.ParseIP(ipStr).To4()
    fmt.Printf("net.IP bytes: %v\n", ip)  // [10 0 0 1]

    // net.IP -> string
    backToStr := ip.String()
    fmt.Printf("Back to string: %s\n", backToStr)  // "10.0.0.1"

    // Using net.IPv4() to construct from octets
    constructed := net.IPv4(192, 168, 1, 1)
    fmt.Printf("Constructed: %s\n", constructed)  // "192.168.1.1"
}
```

## Comparing IPv4 Addresses

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    ip1 := net.ParseIP("192.168.1.1").To4()
    ip2 := net.ParseIP("192.168.1.1").To4()
    ip3 := net.ParseIP("10.0.0.1").To4()

    // net.IP.Equal() compares two IPs regardless of representation (4-byte vs 16-byte)
    fmt.Println(ip1.Equal(ip2))  // true
    fmt.Println(ip1.Equal(ip3))  // false

    // Check if it's a loopback
    fmt.Println(net.ParseIP("127.0.0.1").IsLoopback())  // true

    // Check if it's a private address
    fmt.Println(net.ParseIP("192.168.1.1").IsPrivate())  // true (Go 1.17+)

    // Check if it's a multicast
    fmt.Println(net.ParseIP("224.0.0.1").IsMulticast())  // true
}
```

## Working with IPv4 in Byte Slices

```go
package main

import (
    "encoding/binary"
    "fmt"
    "net"
)

func ipToUint32(ip net.IP) uint32 {
    ip4 := ip.To4()
    if ip4 == nil {
        return 0
    }
    return binary.BigEndian.Uint32(ip4)
}

func uint32ToIP(n uint32) net.IP {
    ip := make(net.IP, 4)
    binary.BigEndian.PutUint32(ip, n)
    return ip
}

func main() {
    ip := net.ParseIP("192.168.1.1")
    num := ipToUint32(ip)
    fmt.Printf("192.168.1.1 as uint32: %d\n", num)

    back := uint32ToIP(num)
    fmt.Printf("uint32 back to IP: %s\n", back)
}
```

## Conclusion

`net.ParseIP` is the standard entry point for parsing IPv4 (and IPv6) addresses in Go. Always call `.To4()` to get a 4-byte representation and to confirm the address is IPv4. Use `.Equal()` for comparison, and `.IsLoopback()`, `.IsPrivate()`, `.IsMulticast()` for classification.
