# How to Use Go net.ListenConfig to Customize IPv4 Socket Options

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, net.ListenConfig, IPv4, Socket Options, Networking, TCP, syscall

Description: Use Go's net.ListenConfig with a Control function to set low-level IPv4 socket options like SO_REUSEPORT, TCP_FASTOPEN, and SO_KEEPALIVE before binding a listener.

## Introduction

Go's standard `net.Listen` creates a TCP listener with sensible defaults, but sometimes you need to customize socket options — for example, enabling `SO_REUSEPORT` for multi-process servers, `TCP_FASTOPEN` for reduced latency, or `SO_RCVBUF`/`SO_SNDBUF` for high-throughput servers. `net.ListenConfig` with a `Control` function provides this capability.

## Basic ListenConfig Usage

```go
package main

import (
    "context"
    "fmt"
    "net"
    "syscall"
)

func main() {
    // Create a ListenConfig with a Control function
    lc := net.ListenConfig{
        Control: func(network, address string, conn syscall.RawConn) error {
            var setsockoptErr error
            
            err := conn.Control(func(fd uintptr) {
                // Enable SO_REUSEPORT: allow multiple sockets to bind to the same port
                // Useful for multi-process servers (e.g., running multiple Go processes)
                setsockoptErr = syscall.SetsockoptInt(
                    int(fd),
                    syscall.SOL_SOCKET,
                    syscall.SO_REUSEPORT,
                    1,
                )
            })
            
            if err != nil {
                return err
            }
            return setsockoptErr
        },
    }

    // Create the listener using the configured options
    ln, err := lc.Listen(context.Background(), "tcp4", ":8080")
    if err != nil {
        panic(err)
    }
    defer ln.Close()

    fmt.Println("Listening on :8080 with SO_REUSEPORT enabled")
    // Accept connections...
}
```

## Setting Multiple Socket Options

```go
package main

import (
    "context"
    "fmt"
    "net"
    "syscall"
)

func createOptimizedListener(address string) (net.Listener, error) {
    lc := net.ListenConfig{
        Control: func(network, address string, conn syscall.RawConn) error {
            return conn.Control(func(fd uintptr) {
                fdInt := int(fd)
                
                // SO_REUSEADDR: allow rebinding to a recently used port
                syscall.SetsockoptInt(fdInt, syscall.SOL_SOCKET, syscall.SO_REUSEADDR, 1)
                
                // SO_REUSEPORT: allow multiple listeners on the same port (Linux)
                syscall.SetsockoptInt(fdInt, syscall.SOL_SOCKET, syscall.SO_REUSEPORT, 1)
                
                // TCP_NODELAY: disable Nagle's algorithm (reduce latency for small messages)
                syscall.SetsockoptInt(fdInt, syscall.IPPROTO_TCP, syscall.TCP_NODELAY, 1)
                
                // SO_RCVBUF: increase receive buffer (bytes) for high-throughput servers
                syscall.SetsockoptInt(fdInt, syscall.SOL_SOCKET, syscall.SO_RCVBUF, 4*1024*1024)
                
                // SO_SNDBUF: increase send buffer
                syscall.SetsockoptInt(fdInt, syscall.SOL_SOCKET, syscall.SO_SNDBUF, 4*1024*1024)
            })
        },
        // Set keepalive interval (Go 1.20+)
        KeepAlive: 30e9, // 30 seconds in nanoseconds
    }

    return lc.Listen(context.Background(), "tcp4", address)
}
```

## Enabling TCP Keep-Alive

```go
lc := net.ListenConfig{
    // Built-in keepalive support (no raw socket required)
    KeepAlive: 30 * time.Second,
}

ln, err := lc.Listen(context.Background(), "tcp4", ":9090")
```

For more control over keep-alive parameters (idle time, interval, probes), use the Control function:

```go
import "golang.org/x/sys/unix"

Control: func(network, address string, conn syscall.RawConn) error {
    return conn.Control(func(fd uintptr) {
        fdInt := int(fd)
        // Enable TCP keepalive
        unix.SetsockoptInt(fdInt, unix.SOL_SOCKET, unix.SO_KEEPALIVE, 1)
        // Idle time before first keepalive probe (seconds)
        unix.SetsockoptInt(fdInt, unix.IPPROTO_TCP, unix.TCP_KEEPIDLE, 60)
        // Interval between probes (seconds)
        unix.SetsockoptInt(fdInt, unix.IPPROTO_TCP, unix.TCP_KEEPINTVL, 10)
        // Number of probes before declaring dead
        unix.SetsockoptInt(fdInt, unix.IPPROTO_TCP, unix.TCP_KEEPCNT, 5)
    })
},
```

## Dialing with Custom Socket Options

Use `net.Dialer.Control` for the client side:

```go
dialer := &net.Dialer{
    Timeout:   5 * time.Second,
    KeepAlive: 30 * time.Second,
    Control: func(network, address string, conn syscall.RawConn) error {
        return conn.Control(func(fd uintptr) {
            // Set IP TOS for QoS marking on client connections
            syscall.SetsockoptInt(int(fd), syscall.IPPROTO_IP, syscall.IP_TOS, 0x10) // Minimize delay
        })
    },
}

conn, err := dialer.DialContext(context.Background(), "tcp4", "10.0.0.1:8080")
```

## Conclusion

`net.ListenConfig` and `net.Dialer.Control` give you full control over IPv4 socket options in Go without dropping down to CGO or raw system calls. Use them to tune buffer sizes, enable SO_REUSEPORT for multi-process serving, or configure keepalive parameters for your specific workload.
