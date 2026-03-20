# How to Build a Concurrent TCP Server in Go with IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, TCP, Concurrency, Goroutines, IPv4, Server, Networking

Description: Learn how to build a high-concurrency IPv4 TCP server in Go using goroutines, with connection tracking, graceful shutdown, and resource management.

## Go's Concurrency Model for TCP

Go's goroutines are extremely lightweight (starting at ~8KB stack) making the "goroutine per connection" pattern practical even for thousands of concurrent clients—unlike threads in other languages.

## Production-Grade Concurrent TCP Server

```go
package main

import (
    "context"
    "io"
    "log"
    "net"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

type Server struct {
    listener net.Listener
    wg       sync.WaitGroup
    quit     chan struct{}
}

func NewServer(addr string) (*Server, error) {
    ln, err := net.Listen("tcp4", addr)
    if err != nil {
        return nil, err
    }
    return &Server{
        listener: ln,
        quit:     make(chan struct{}),
    }, nil
}

func (s *Server) handleConn(conn net.Conn) {
    defer func() {
        conn.Close()
        s.wg.Done()
    }()

    addr := conn.RemoteAddr().String()
    log.Printf("[%s] Connected", addr)

    buf := make([]byte, 4096)
    for {
        // Set read deadline to detect idle clients
        conn.SetReadDeadline(time.Now().Add(60 * time.Second))

        n, err := conn.Read(buf)
        if err != nil {
            if err == io.EOF {
                log.Printf("[%s] Disconnected gracefully", addr)
            } else if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
                log.Printf("[%s] Idle timeout", addr)
            } else {
                log.Printf("[%s] Read error: %v", addr, err)
            }
            return
        }

        // Echo data back
        conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
        if _, err := conn.Write(buf[:n]); err != nil {
            log.Printf("[%s] Write error: %v", addr, err)
            return
        }
    }
}

func (s *Server) Serve() {
    for {
        conn, err := s.listener.Accept()
        if err != nil {
            select {
            case <-s.quit:
                return  // Shutdown requested
            default:
                log.Printf("Accept error: %v", err)
            }
            continue
        }

        s.wg.Add(1)
        go s.handleConn(conn)
    }
}

func (s *Server) Shutdown() {
    close(s.quit)
    s.listener.Close()

    // Wait for all connections to finish (with timeout)
    done := make(chan struct{})
    go func() {
        s.wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        log.Println("All connections closed cleanly")
    case <-time.After(30 * time.Second):
        log.Println("Shutdown timeout: some connections may have been dropped")
    }
}

func main() {
    srv, err := NewServer("0.0.0.0:9000")
    if err != nil {
        log.Fatalf("Failed to start server: %v", err)
    }

    log.Printf("Server listening on %s", srv.listener.Addr())

    // Start serving in a goroutine
    go srv.Serve()

    // Wait for SIGINT or SIGTERM
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    <-ctx.Done()

    log.Println("Shutting down server...")
    srv.Shutdown()
    log.Println("Server stopped")
}
```

## Limiting Concurrency

Prevent resource exhaustion with a semaphore:

```go
type Server struct {
    listener   net.Listener
    wg         sync.WaitGroup
    semaphore  chan struct{}   // Limits concurrent connections
}

func NewServerWithLimit(addr string, maxConns int) (*Server, error) {
    ln, err := net.Listen("tcp4", addr)
    if err != nil {
        return nil, err
    }
    return &Server{
        listener:  ln,
        semaphore: make(chan struct{}, maxConns),
    }, nil
}

func (s *Server) Serve() {
    for {
        conn, err := s.listener.Accept()
        if err != nil {
            break
        }

        // Acquire semaphore (blocks if at max connections)
        s.semaphore <- struct{}{}
        s.wg.Add(1)

        go func(c net.Conn) {
            defer func() {
                <-s.semaphore  // Release slot
                s.wg.Done()
                c.Close()
            }()
            // Handle connection...
        }(conn)
    }
}
```

## Conclusion

Go's goroutine-per-connection pattern is idiomatic and scales well. Use `sync.WaitGroup` to track active connections for graceful shutdown, set read/write deadlines to clean up idle clients, and a semaphore channel to cap maximum concurrency. Closing the listener causes `Accept()` to return an error, cleanly stopping the accept loop.
