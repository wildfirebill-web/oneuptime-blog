# How to Implement Connection Pooling for IPv4 TCP in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, Connection Pooling, IPv4, TCP, Performance, Networking

Description: Build a reusable TCP connection pool in Go to efficiently manage IPv4 connections, reducing overhead from repeated connection establishment to the same backend.

## Introduction

Establishing a TCP connection involves a three-way handshake and TLS negotiation, adding 1–50ms of latency per request. For high-frequency operations to the same backend (database, cache, microservice), connection pooling amortizes this cost by reusing established connections.

## Simple Connection Pool Implementation

```go
package main

import (
    "errors"
    "fmt"
    "net"
    "sync"
    "time"
)

// Pool represents a thread-safe pool of TCP connections
type Pool struct {
    mu       sync.Mutex
    conns    chan net.Conn
    address  string
    maxSize  int
    timeout  time.Duration
}

// NewPool creates a connection pool for the given IPv4 address
func NewPool(address string, initialSize, maxSize int, timeout time.Duration) (*Pool, error) {
    p := &Pool{
        conns:   make(chan net.Conn, maxSize),
        address: address,
        maxSize: maxSize,
        timeout: timeout,
    }

    // Pre-fill with initial connections
    for i := 0; i < initialSize; i++ {
        conn, err := net.DialTimeout("tcp4", address, timeout)
        if err != nil {
            p.Close()
            return nil, fmt.Errorf("failed to initialize pool: %w", err)
        }
        p.conns <- conn
    }

    return p, nil
}

// Get retrieves a connection from the pool, creating a new one if needed
func (p *Pool) Get() (net.Conn, error) {
    select {
    case conn := <-p.conns:
        // Check if the connection is still alive
        if err := p.checkConn(conn); err != nil {
            conn.Close()
            // Create a fresh connection to replace the dead one
            return net.DialTimeout("tcp4", p.address, p.timeout)
        }
        return conn, nil
    default:
        // Pool empty — create a new connection (up to maxSize in flight)
        return net.DialTimeout("tcp4", p.address, p.timeout)
    }
}

// Put returns a connection to the pool
func (p *Pool) Put(conn net.Conn) {
    if conn == nil {
        return
    }
    select {
    case p.conns <- conn:
        // Returned to pool successfully
    default:
        // Pool is full — close the excess connection
        conn.Close()
    }
}

// checkConn verifies a connection is still alive with a zero-byte write
func (p *Pool) checkConn(conn net.Conn) error {
    conn.SetDeadline(time.Now().Add(100 * time.Millisecond))
    _, err := conn.Write([]byte{})
    conn.SetDeadline(time.Time{})   // Clear deadline
    return err
}

// Close drains and closes all pooled connections
func (p *Pool) Close() {
    close(p.conns)
    for conn := range p.conns {
        conn.Close()
    }
}
```

## Using the Connection Pool

```go
package main

import (
    "bufio"
    "fmt"
    "sync"
    "time"
)

func main() {
    // Create a pool with 5 initial connections, max 20
    pool, err := NewPool("10.0.0.10:6379", 5, 20, 5*time.Second)
    if err != nil {
        panic(err)
    }
    defer pool.Close()

    // Simulate 100 concurrent requests using the pool
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(reqNum int) {
            defer wg.Done()

            // Get a connection from the pool
            conn, err := pool.Get()
            if err != nil {
                fmt.Printf("Request %d: failed to get connection: %v\n", reqNum, err)
                return
            }
            
            // Important: return the connection to the pool when done
            defer pool.Put(conn)

            // Use the connection
            conn.SetDeadline(time.Now().Add(5 * time.Second))
            fmt.Fprintf(conn, "PING\r\n")
            
            reader := bufio.NewReader(conn)
            response, err := reader.ReadString('\n')
            if err != nil {
                fmt.Printf("Request %d: read error: %v\n", reqNum, err)
                pool.Put(conn)   // Still put back — checkConn will detect if dead
                return
            }
            
            fmt.Printf("Request %d: %s", reqNum, response)
        }(i)
    }

    wg.Wait()
    fmt.Println("All requests completed")
}
```

## Using sync.Pool for Short-Lived Buffers

For request/response buffers (not connections), use `sync.Pool`:

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)   // 4KB buffer
    },
}

// In your handler:
buf := bufPool.Get().([]byte)
defer bufPool.Put(buf)  // Return buffer to pool when done
```

## Using net/http's Built-In Pool

For HTTP workloads, `http.Client` has a built-in connection pool via `http.Transport`:

```go
transport := &http.Transport{
    MaxIdleConns:        100,              // Maximum idle (keepalive) connections
    MaxIdleConnsPerHost: 10,              // Per-host idle connections
    IdleConnTimeout:     90 * time.Second,
    DialContext: (&net.Dialer{
        Timeout:   30 * time.Second,
        KeepAlive: 30 * time.Second,
        // Force IPv4
        Control: func(network, address string, c syscall.RawConn) error {
            return nil
        },
    }).DialContext,
}

client := &http.Client{Transport: transport}
```

## Conclusion

Connection pooling is essential for high-throughput Go services that communicate with backends over IPv4 TCP. The custom pool shown here handles connection validation and replacement. For HTTP workloads, configure `http.Transport` instead of building a custom pool.
