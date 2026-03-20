# How to Configure Gin (Go) for IPv6 Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Gin, Go, Golang, IPv6, Web Framework, Dual-Stack, Net/http

Description: Configure the Gin web framework to listen on IPv6, extract client IPv6 addresses, and handle IPv6 in middleware using Go's standard net/netip package.

## Introduction

Gin uses Go's `net/http` package under the hood. Listening on IPv6 requires passing an IPv6 address or `[::]:port` as the bind address to `gin.Run()` or a custom `http.Server`. Go's `net/netip` package (1.18+) provides efficient IPv6 address parsing.

## Step 1: Listen on IPv6

```go
// main.go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    r.GET("/", func(c *gin.Context) {
        c.JSON(200, gin.H{"hello": "ipv6"})
    })

    // Listen on all IPv6 interfaces (dual-stack on Linux)
    r.Run("[::]:8080")
}
```

```go
// With custom server for IPv6-only
func main() {
    r := gin.Default()

    server := &http.Server{
        Addr:    "[::]:8080",
        Handler: r,
    }

    // For IPv6-only (no IPv4 dual-stack)
    ln, err := net.Listen("tcp6", "[::]:8080")
    if err != nil {
        panic(err)
    }
    server.Serve(ln)
}
```

## Step 2: Get Client IPv6 Address

```go
// middleware/client_ip.go
package middleware

import (
    "net"
    "net/netip"
    "strings"

    "github.com/gin-gonic/gin"
)

func ClientIPMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        ip := extractClientIP(c)
        c.Set("client_ip", ip)
        c.Next()
    }
}

func extractClientIP(c *gin.Context) string {
    // Check X-Forwarded-For
    xff := c.GetHeader("X-Forwarded-For")
    if xff != "" {
        ip := strings.TrimSpace(strings.Split(xff, ",")[0])
        if addr, err := netip.ParseAddr(ip); err == nil {
            return addr.String()
        }
    }

    // Fall back to remote address
    host, _, err := net.SplitHostPort(c.Request.RemoteAddr)
    if err != nil {
        return c.Request.RemoteAddr
    }

    // Normalize IPv4-mapped IPv6
    if addr, err := netip.ParseAddr(host); err == nil {
        if addr.Is4In6() {
            return addr.Unmap().String()
        }
        return addr.String()
    }

    return host
}
```

## Step 3: IPv6 Address Validation

```go
// handlers/network.go
package handlers

import (
    "net/netip"
    "github.com/gin-gonic/gin"
)

type EndpointRequest struct {
    Address string `json:"address" binding:"required"`
    Port    int    `json:"port" binding:"required,min=1,max=65535"`
}

func CreateEndpoint(c *gin.Context) {
    var req EndpointRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }

    addr, err := netip.ParseAddr(req.Address)
    if err != nil {
        c.JSON(400, gin.H{"error": "invalid IP address"})
        return
    }

    if addr.IsLoopback() {
        c.JSON(400, gin.H{"error": "loopback addresses not allowed"})
        return
    }

    var url string
    if addr.Is6() && !addr.Is4In6() {
        url = fmt.Sprintf("http://[%s]:%d", addr.String(), req.Port)
    } else {
        url = fmt.Sprintf("http://%s:%d", addr.Unmap().String(), req.Port)
    }

    c.JSON(200, gin.H{
        "address":  addr.String(),
        "is_ipv6":  addr.Is6() && !addr.Is4In6(),
        "url":      url,
    })
}
```

## Step 4: Rate Limiting by /64 Subnet

```go
// middleware/rate_limit.go
package middleware

import (
    "net/netip"
    "sync"
    "time"
    "github.com/gin-gonic/gin"
)

type RateLimiter struct {
    mu       sync.Mutex
    counters map[string][]time.Time
    limit    int
    window   time.Duration
}

func (rl *RateLimiter) getKey(ip string) string {
    addr, err := netip.ParseAddr(ip)
    if err != nil || !addr.Is6() || addr.Is4In6() {
        return ip
    }
    // Rate limit by /64
    prefix, _ := addr.Prefix(64)
    return prefix.String()
}

func (rl *RateLimiter) Allow(ip string) bool {
    key := rl.getKey(ip)
    rl.mu.Lock()
    defer rl.mu.Unlock()
    now := time.Now()
    // filter old entries
    filtered := rl.counters[key][:0]
    for _, t := range rl.counters[key] {
        if now.Sub(t) < rl.window {
            filtered = append(filtered, t)
        }
    }
    filtered = append(filtered, now)
    rl.counters[key] = filtered
    return len(filtered) <= rl.limit
}
```

## Step 5: Test

```bash
go run main.go

# Test IPv6

curl -6 http://[::1]:8080/
curl -6 http://[2001:db8::1]:8080/endpoint \
    -d '{"address":"2001:db8::42","port":443}'

# Verify listening
ss -lntp | grep :8080
```

## Conclusion

Gin on IPv6 uses `r.Run("[::]:8080")` or a custom `net.Listen("tcp6", ...)` for IPv6-only mode. Use `net/netip.ParseAddr` for efficient, allocation-free IPv6 address parsing. The `Is4In6()` method identifies IPv4-mapped addresses for normalization. Rate-limit by /64 prefixes using `addr.Prefix(64)`. Monitor Gin endpoints with OneUptime's IPv6 HTTP checks.
