# How to Use net.Dial with IPv4-Only Connections in Go (tcp4)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, net.Dial, IPv4, tcp4, Networking, Dual-Stack

Description: Learn how to force IPv4-only TCP connections in Go using the tcp4 network type with net.Dial and net.Dialer to avoid dual-stack ambiguity.

## Why Force IPv4?

By default, `net.Dial("tcp", ...)` may use IPv6 if the target resolves to a dual-stack host. Explicitly using `"tcp4"` forces Go's network stack to use IPv4 only—useful for environments where IPv6 is not fully configured or for compliance with IPv4-only infrastructure.

## Using "tcp4" Network Type

```go
package main

import (
    "fmt"
    "log"
    "net"
)

func main() {
    // "tcp4" explicitly selects IPv4 TCP
    // "tcp6" would force IPv6; "tcp" allows either
    conn, err := net.Dial("tcp4", "google.com:80")
    if err != nil {
        log.Fatalf("Failed to connect via IPv4: %v", err)
    }
    defer conn.Close()

    // Verify we have an IPv4 connection
    localAddr := conn.LocalAddr().(*net.TCPAddr)
    remoteAddr := conn.RemoteAddr().(*net.TCPAddr)

    fmt.Printf("Local:  %s (IPv4: %v)\n", localAddr, localAddr.IP.To4() != nil)
    fmt.Printf("Remote: %s (IPv4: %v)\n", remoteAddr, remoteAddr.IP.To4() != nil)

    // Send a minimal HTTP request
    fmt.Fprintf(conn, "GET / HTTP/1.1\r\nHost: google.com\r\nConnection: close\r\n\r\n")

    buf := make([]byte, 4096)
    n, _ := conn.Read(buf)
    fmt.Printf("Response:\n%s\n", buf[:n])
}
```

## net.Dialer with IPv4 Control

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net"
    "time"
)

func dialIPv4Only(ctx context.Context, host, port string) (net.Conn, error) {
    dialer := &net.Dialer{
        Timeout:   10 * time.Second,
        KeepAlive: 30 * time.Second,
    }
    // Use "tcp4" to restrict to IPv4
    return dialer.DialContext(ctx, "tcp4", net.JoinHostPort(host, port))
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    conn, err := dialIPv4Only(ctx, "8.8.8.8", "53")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    fmt.Printf("Connected to %s via %s\n", conn.RemoteAddr(), conn.LocalAddr())
}
```

## Listening on IPv4-Only

```go
package main

import (
    "log"
    "net"
)

func main() {
    // "tcp4" on the listener also restricts to IPv4
    ln, err := net.Listen("tcp4", ":8080")
    if err != nil {
        log.Fatal(err)
    }
    defer ln.Close()

    log.Printf("Listening on IPv4: %s", ln.Addr())

    for {
        conn, err := ln.Accept()
        if err != nil {
            break
        }
        go func(c net.Conn) {
            defer c.Close()
            // Handle connection
            addr := c.RemoteAddr().(*net.TCPAddr)
            log.Printf("Client: %s (IPv4: %v)", addr, addr.IP.To4() != nil)
        }(conn)
    }
}
```

## Resolving to IPv4 Before Dialing

When you want to dial a hostname but ensure IPv4:

```go
package main

import (
    "fmt"
    "log"
    "net"
)

func resolveAndDialIPv4(hostname, port string) (net.Conn, error) {
    // Resolve to IPv4 first
    ips, err := net.LookupIP(hostname)
    if err != nil {
        return nil, err
    }

    var ipv4 net.IP
    for _, ip := range ips {
        if v4 := ip.To4(); v4 != nil {
            ipv4 = v4
            break
        }
    }

    if ipv4 == nil {
        return nil, fmt.Errorf("no IPv4 address found for %s", hostname)
    }

    // Dial the explicit IPv4 address
    return net.Dial("tcp4", net.JoinHostPort(ipv4.String(), port))
}

func main() {
    conn, err := resolveAndDialIPv4("example.com", "443")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()
    log.Printf("Connected: %s -> %s", conn.LocalAddr(), conn.RemoteAddr())
}
```

## Network Type Summary

| Network | Behavior |
|---------|---------|
| `"tcp"` | IPv4 or IPv6, OS chooses |
| `"tcp4"` | IPv4 only |
| `"tcp6"` | IPv6 only |
| `"udp4"` | UDP over IPv4 only |
| `"udp6"` | UDP over IPv6 only |

## Conclusion

Using `"tcp4"` with `net.Dial`, `net.Listen`, and `net.Dialer.DialContext` explicitly restricts network operations to IPv4 in Go. This is essential in dual-stack environments where you need predictable IPv4 behavior. For hostnames, resolve to IPv4 first with `net.LookupIP` and filter with `.To4() != nil`.
