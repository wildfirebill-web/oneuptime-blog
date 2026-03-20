# How to Validate IPv4 Addresses in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, IPv4, Validation, net package, Networking

Description: Learn multiple ways to validate IPv4 addresses in Go using net.ParseIP, net.ParseCIDR, and custom validation logic.

## Method 1: Using net.ParseIP

```go
package main

import (
    "fmt"
    "net"
)

// IsValidIPv4 returns true if the string is a valid IPv4 address
func IsValidIPv4(s string) bool {
    ip := net.ParseIP(s)
    return ip != nil && ip.To4() != nil
}

func main() {
    tests := []string{
        "192.168.1.1",     // valid
        "10.0.0.0",        // valid (network address, still valid IP)
        "255.255.255.255", // valid (broadcast)
        "0.0.0.0",         // valid
        "256.0.0.1",       // invalid (octet > 255)
        "192.168.1",       // invalid (missing octet)
        "::1",             // invalid (IPv6)
        "",                // invalid (empty)
        "192.168.1.1/24",  // invalid (CIDR notation, not bare IP)
    }

    for _, t := range tests {
        fmt.Printf("%-22s -> %v\n", t, IsValidIPv4(t))
    }
}
```

## Method 2: Validating Private vs Public

```go
package main

import (
    "fmt"
    "net"
)

var privateRanges = []net.IPNet{
    {IP: net.ParseIP("10.0.0.0"), Mask: net.CIDRMask(8, 32)},
    {IP: net.ParseIP("172.16.0.0"), Mask: net.CIDRMask(12, 32)},
    {IP: net.ParseIP("192.168.0.0"), Mask: net.CIDRMask(16, 32)},
    {IP: net.ParseIP("127.0.0.0"), Mask: net.CIDRMask(8, 32)},
}

func IsPrivateIPv4(s string) bool {
    ip := net.ParseIP(s)
    if ip == nil || ip.To4() == nil {
        return false
    }
    for _, r := range privateRanges {
        if r.Contains(ip) {
            return true
        }
    }
    return false
}

func main() {
    fmt.Println(IsPrivateIPv4("192.168.1.50"))  // true
    fmt.Println(IsPrivateIPv4("10.0.0.1"))      // true
    fmt.Println(IsPrivateIPv4("8.8.8.8"))       // false
}
```

## Method 3: Validate and Classify

```go
package main

import (
    "fmt"
    "net"
)

type IPv4Classification struct {
    Valid     bool
    Loopback  bool
    Private   bool
    Multicast bool
    Broadcast bool
    Global    bool
}

func ClassifyIPv4(s string) IPv4Classification {
    ip := net.ParseIP(s)
    if ip == nil || ip.To4() == nil {
        return IPv4Classification{}
    }

    return IPv4Classification{
        Valid:     true,
        Loopback:  ip.IsLoopback(),
        Private:   ip.IsPrivate(),   // Go 1.17+
        Multicast: ip.IsMulticast(),
        Broadcast: ip.Equal(net.IPv4bcast),
        Global:    ip.IsGlobalUnicast(),
    }
}

func main() {
    for _, ip := range []string{"8.8.8.8", "192.168.1.1", "127.0.0.1", "224.0.0.1"} {
        c := ClassifyIPv4(ip)
        fmt.Printf("%s: valid=%v private=%v loopback=%v multicast=%v global=%v\n",
            ip, c.Valid, c.Private, c.Loopback, c.Multicast, c.Global)
    }
}
```

## Method 4: Validating with Regex (Simple Cases)

```go
package main

import (
    "fmt"
    "regexp"
)

// Regex for validating IPv4 addresses (doesn't check octet range)
var ipv4Regex = regexp.MustCompile(
    `^(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})$`,
)

// For production, prefer net.ParseIP which handles all edge cases
func IsIPv4Format(s string) bool {
    return ipv4Regex.MatchString(s)
}
```

**Note**: Regex alone doesn't validate octet ranges (0-255). Always use `net.ParseIP` for authoritative validation.

## Validating User Input

```go
package main

import (
    "errors"
    "fmt"
    "net"
)

func ParseAndValidateIPv4(input string) (net.IP, error) {
    if input == "" {
        return nil, errors.New("IP address cannot be empty")
    }

    ip := net.ParseIP(input)
    if ip == nil {
        return nil, fmt.Errorf("'%s' is not a valid IP address", input)
    }

    ip4 := ip.To4()
    if ip4 == nil {
        return nil, fmt.Errorf("'%s' is IPv6; expected IPv4", input)
    }

    return ip4, nil
}

func main() {
    inputs := []string{"192.168.1.1", "::1", "999.0.0.1", ""}
    for _, input := range inputs {
        ip, err := ParseAndValidateIPv4(input)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
        } else {
            fmt.Printf("Valid IPv4: %s\n", ip)
        }
    }
}
```

## Conclusion

The most reliable way to validate IPv4 addresses in Go is `net.ParseIP(s).To4() != nil`. It handles all edge cases including leading zeros, range validation, and distinguishing IPv4 from IPv6. Avoid pure regex validation as it can't enforce the 0–255 octet range constraint.
