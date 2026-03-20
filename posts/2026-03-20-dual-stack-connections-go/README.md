# How to Handle Dual-Stack Connections in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, IPv6, Dual-Stack, TCP, Happy Eyeballs, Networking

Description: Handle dual-stack (IPv4 and IPv6) connections in Go servers and clients, including Happy Eyeballs implementation and address preference configuration.

## Dual-Stack Server

A dual-stack server accepts connections over both IPv4 and IPv6:

```go
package main

import (
    "context"
    "fmt"
    "net"
    "net/netip"
    "sync"
)

type DualStackServer struct {
    v4Listener net.Listener
    v6Listener net.Listener
}

func NewDualStackServer(port int) (*DualStackServer, error) {
    v4Addr := fmt.Sprintf("0.0.0.0:%d", port)
    v6Addr := fmt.Sprintf("[::]:%d", port)

    v4ln, err := net.Listen("tcp4", v4Addr)
    if err != nil {
        return nil, fmt.Errorf("IPv4 listen failed: %w", err)
    }

    v6ln, err := net.Listen("tcp6", v6Addr)
    if err != nil {
        v4ln.Close()
        return nil, fmt.Errorf("IPv6 listen failed: %w", err)
    }

    return &DualStackServer{v4ln, v6ln}, nil
}

func (s *DualStackServer) Accept() (net.Conn, error) {
    // Accept from either listener using channels
    connCh := make(chan net.Conn, 2)
    errCh := make(chan error, 2)

    accept := func(ln net.Listener) {
        conn, err := ln.Accept()
        if err != nil {
            errCh <- err
        } else {
            connCh <- conn
        }
    }

    go accept(s.v4Listener)
    go accept(s.v6Listener)

    select {
    case conn := <-connCh:
        return conn, nil
    case err := <-errCh:
        return nil, err
    }
}

func getIPVersion(conn net.Conn) string {
    remoteAddr := conn.RemoteAddr().(*net.TCPAddr)
    ip, ok := netip.AddrFromSlice(remoteAddr.IP)
    if !ok {
        return "unknown"
    }
    if ip.Is4() || ip.Is4In6() {
        return "IPv4"
    }
    return "IPv6"
}
```

## Happy Eyeballs Client (RFC 8305)

Happy Eyeballs connects using whichever IPv4/IPv6 succeeds first:

```go
package main

import (
    "context"
    "fmt"
    "net"
    "time"
)

// DialHappyEyeballs connects to a host preferring IPv6 but falling back to IPv4.
// It implements a simplified version of Happy Eyeballs (RFC 8305).
func DialHappyEyeballs(ctx context.Context, host, port string) (net.Conn, error) {
    // Resolve all addresses
    addrs, err := net.DefaultResolver.LookupIPAddr(ctx, host)
    if err != nil {
        return nil, fmt.Errorf("lookup failed: %w", err)
    }

    // Separate into IPv6 and IPv4
    var v6, v4 []net.IPAddr
    for _, addr := range addrs {
        if addr.IP.To4() == nil {
            v6 = append(v6, addr)
        } else {
            v4 = append(v4, addr)
        }
    }

    type result struct {
        conn net.Conn
        err  error
    }

    resultCh := make(chan result, len(v6)+len(v4))

    dialer := &net.Dialer{Timeout: 10 * time.Second}

    // Try IPv6 first
    for _, addr := range v6 {
        go func(ip net.IPAddr) {
            target := net.JoinHostPort(ip.IP.String(), port)
            conn, err := dialer.DialContext(ctx, "tcp6", target)
            resultCh <- result{conn, err}
        }(addr)
    }

    // Delay IPv4 attempts by 250ms (Happy Eyeballs delay)
    go func() {
        timer := time.NewTimer(250 * time.Millisecond)
        defer timer.Stop()
        select {
        case <-timer.C:
            for _, addr := range v4 {
                go func(ip net.IPAddr) {
                    target := net.JoinHostPort(ip.IP.String(), port)
                    conn, err := dialer.DialContext(ctx, "tcp4", target)
                    resultCh <- result{conn, err}
                }(ip)
            }
        case <-ctx.Done():
        }
    }()

    // Return the first successful connection
    var lastErr error
    for i := 0; i < len(v6)+len(v4); i++ {
        r := <-resultCh
        if r.err == nil {
            return r.conn, nil
        }
        lastErr = r.err
    }

    return nil, fmt.Errorf("all connections failed, last error: %w", lastErr)
}
```

## Using net.Dialer for IPv6 Preference

```go
package main

import (
    "context"
    "net"
    "net/http"
    "time"
)

// createIPv6PreferringTransport creates an HTTP transport that prefers IPv6.
func createIPv6PreferringTransport() *http.Transport {
    dialer := &net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
        // Control function to prefer IPv6
        Control: nil,  // Could add net.Control for advanced options
    }

    return &http.Transport{
        DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
            host, port, err := net.SplitHostPort(addr)
            if err != nil {
                return nil, err
            }

            // Try to connect via Happy Eyeballs
            return DialHappyEyeballs(ctx, host, port)
        },
        MaxIdleConns:       100,
        IdleConnTimeout:    90 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
    }
}

func main() {
    client := &http.Client{
        Transport: createIPv6PreferringTransport(),
        Timeout:   30 * time.Second,
    }

    resp, err := client.Get("https://example.com")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()
    fmt.Println("Status:", resp.Status)
}
```

## Detecting Connection IP Version at Runtime

```go
import "net/netip"

func getConnectionIPVersion(conn net.Conn) int {
    tcpAddr, ok := conn.RemoteAddr().(*net.TCPAddr)
    if !ok {
        return 0
    }

    ip, ok := netip.AddrFromSlice(tcpAddr.IP)
    if !ok {
        return 0
    }

    // IPv4-mapped IPv6 is effectively IPv4
    if ip.Is4() || ip.Is4In6() {
        return 4
    }
    return 6
}
```

## Conclusion

Dual-stack support in Go requires either a single `[::]` listener (dual-stack via IPv4-mapped) or separate IPv4 and IPv6 listeners. For clients, implement Happy Eyeballs (RFC 8305) to prefer IPv6 while falling back to IPv4 after a 250ms delay. Go's standard `net.Dialer` works with both address families, making dual-stack client code straightforward.
