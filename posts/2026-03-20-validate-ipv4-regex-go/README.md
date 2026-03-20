# How to Validate IPv4 Addresses Using Regex in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, Regex, IPv4, Validation, regexp, Networking

Description: Learn how to validate IPv4 address strings using regular expressions in Go, using a strict octet-range pattern compiled once with the regexp package.

## Simple Pattern (Not Recommended)

```go
package main

import (
    "fmt"
    "regexp"
)

// Naive: accepts 999.999.999.999 — does not check octet range
var naiveIPv4 = regexp.MustCompile(`^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$`)

func main() {
    fmt.Println(naiveIPv4.MatchString("192.168.1.1"))    // true
    fmt.Println(naiveIPv4.MatchString("999.999.999.999")) // true — WRONG
}
```

## Strict Regex (Validates 0-255)

```go
package main

import (
    "fmt"
    "regexp"
)

// Compile once at package level — regexp.Regexp is safe for concurrent use
var strictIPv4 = regexp.MustCompile(
    `^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)` +
    `(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$`,
)

func IsValidIPv4(s string) bool {
    return strictIPv4.MatchString(s)
}

func main() {
    cases := []struct {
        ip       string
        expected bool
    }{
        {"192.168.1.1", true},
        {"0.0.0.0", true},
        {"255.255.255.255", true},
        {"256.0.0.1", false},      // Octet > 255
        {"192.168.1", false},      // Missing octet
        {"192.168.1.1.1", false},  // Extra octet
        {"192.168.01.1", false},   // Leading zero
        {"::1", false},            // IPv6
        {"", false},               // Empty
        {" 192.168.1.1", false},   // Leading space
    }

    for _, c := range cases {
        result := IsValidIPv4(c.ip)
        status := "PASS"
        if result != c.expected {
            status = "FAIL"
        }
        fmt.Printf("[%s] %-25q -> %v\n", status, c.ip, result)
    }
}
```

## Extracting IPs from Text

```go
package main

import (
    "fmt"
    "regexp"
)

// FindAll with word boundaries to locate IPs inside log lines
var ipFinder = regexp.MustCompile(
    `\b(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)` +
    `(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}\b`,
)

func ExtractIPs(text string) []string {
    return ipFinder.FindAllString(text, -1)
}

func main() {
    log := "[2026-03-20] Request from 192.168.1.50 forwarded to 10.0.0.1"
    ips := ExtractIPs(log)
    fmt.Println("Found IPs:", ips)  // [192.168.1.50 10.0.0.1]
}
```

## Prefer net.ParseIP for Simple Validation

```go
package main

import (
    "fmt"
    "net"
)

// net.ParseIP is simpler and handles all edge cases
func IsValidIPv4Simple(s string) bool {
    ip := net.ParseIP(s)
    return ip != nil && ip.To4() != nil
}

func main() {
    fmt.Println(IsValidIPv4Simple("192.168.1.1"))   // true
    fmt.Println(IsValidIPv4Simple("256.0.0.1"))     // false
    fmt.Println(IsValidIPv4Simple("::1"))           // false (IPv6)
    fmt.Println(IsValidIPv4Simple("192.168.01.1"))  // false (leading zero)
}
```

## Benchmark: Regex vs net.ParseIP

```go
package main_test

import (
    "net"
    "regexp"
    "testing"
)

var compiled = regexp.MustCompile(
    `^(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)` +
    `(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}$`,
)
const ip = "192.168.100.200"

func BenchmarkRegex(b *testing.B) {
    for i := 0; i < b.N; i++ {
        compiled.MatchString(ip)
    }
}

func BenchmarkParseIP(b *testing.B) {
    for i := 0; i < b.N; i++ {
        p := net.ParseIP(ip)
        _ = p != nil && p.To4() != nil
    }
}
```

## Conclusion

In Go, always compile the `regexp.Regexp` with `regexp.MustCompile` at package level — `*regexp.Regexp` is safe for concurrent use and compilation happens once. The strict alternation pattern correctly validates the 0-255 range and rejects leading zeros. For pure validation (not extraction), prefer `net.ParseIP(s).To4() != nil` which is simpler and typically faster. Use the regex approach only when you need to extract IPs from freeform text.
