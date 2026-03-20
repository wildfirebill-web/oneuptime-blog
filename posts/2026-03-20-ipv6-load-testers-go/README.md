# How to Build IPv6 Load Testers in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Go, Load Testing, Performance, HTTP

Description: Build IPv6 load testing tools in Go to benchmark servers, test dual-stack behavior, measure IPv6 connection latency, and validate IPv6 performance under load.

## Simple IPv6 HTTP Load Tester

```go
package main

import (
    "crypto/tls"
    "flag"
    "fmt"
    "net"
    "net/http"
    "sync"
    "sync/atomic"
    "time"
)

type LoadTestResult struct {
    TotalRequests   int64
    SuccessCount    int64
    ErrorCount      int64
    TotalLatency    int64   // nanoseconds
    MinLatency      int64
    MaxLatency      int64
}

func runIPv6LoadTest(target string, concurrency, requests int, forceIPv6 bool) *LoadTestResult {
    result := &LoadTestResult{MinLatency: 1<<62}

    // Create IPv6-aware HTTP transport
    transport := &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   10 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSClientConfig:     &tls.Config{InsecureSkipVerify: false},
        MaxIdleConns:        concurrency,
        MaxIdleConnsPerHost: concurrency,
    }

    if forceIPv6 {
        // Override dialer to force IPv6
        transport.DialContext = func(ctx interface{ Deadline() (time.Time, bool) },
            network, addr string) (net.Conn, error) {
            d := &net.Dialer{Timeout: 10 * time.Second}
            return d.DialContext(ctx.(interface {
                Deadline() (time.Time, bool)
                Done() <-chan struct{}
                Err() error
                Value(interface{}) interface{}
            }), "tcp6", addr)
        }
    }

    client := &http.Client{
        Transport: transport,
        Timeout:   15 * time.Second,
    }

    var wg sync.WaitGroup
    semaphore := make(chan struct{}, concurrency)

    for i := 0; i < requests; i++ {
        wg.Add(1)
        semaphore <- struct{}{}
        go func() {
            defer wg.Done()
            defer func() { <-semaphore }()

            start := time.Now()
            resp, err := client.Get(target)
            latency := time.Since(start).Nanoseconds()

            atomic.AddInt64(&result.TotalRequests, 1)
            if err != nil || resp.StatusCode >= 500 {
                atomic.AddInt64(&result.ErrorCount, 1)
                if err == nil {
                    resp.Body.Close()
                }
                return
            }
            resp.Body.Close()

            atomic.AddInt64(&result.SuccessCount, 1)
            atomic.AddInt64(&result.TotalLatency, latency)

            // Update min/max
            for {
                old := atomic.LoadInt64(&result.MinLatency)
                if latency >= old || atomic.CompareAndSwapInt64(&result.MinLatency, old, latency) {
                    break
                }
            }
            for {
                old := atomic.LoadInt64(&result.MaxLatency)
                if latency <= old || atomic.CompareAndSwapInt64(&result.MaxLatency, old, latency) {
                    break
                }
            }
        }()
    }

    wg.Wait()
    return result
}

func main() {
    target := flag.String("target", "http://[2001:db8::1]/", "Target URL")
    concurrency := flag.Int("c", 10, "Concurrent connections")
    requests := flag.Int("n", 100, "Total requests")
    ipv6only := flag.Bool("6", true, "Force IPv6 only")
    flag.Parse()

    fmt.Printf("IPv6 Load Test: %s (concurrency=%d, requests=%d)\n",
        *target, *concurrency, *requests)

    start := time.Now()
    result := runIPv6LoadTest(*target, *concurrency, *requests, *ipv6only)
    elapsed := time.Since(start)

    successPct := float64(result.SuccessCount) / float64(result.TotalRequests) * 100
    avgLatency := float64(result.TotalLatency) / float64(result.SuccessCount) / 1e6

    fmt.Printf("\n=== Results ===\n")
    fmt.Printf("Duration:         %.2fs\n", elapsed.Seconds())
    fmt.Printf("Total requests:   %d\n", result.TotalRequests)
    fmt.Printf("Successful:       %d (%.1f%%)\n", result.SuccessCount, successPct)
    fmt.Printf("Errors:           %d\n", result.ErrorCount)
    fmt.Printf("Throughput:       %.1f req/s\n", float64(result.TotalRequests)/elapsed.Seconds())
    fmt.Printf("Avg latency:      %.2f ms\n", avgLatency)
    fmt.Printf("Min latency:      %.2f ms\n", float64(result.MinLatency)/1e6)
    fmt.Printf("Max latency:      %.2f ms\n", float64(result.MaxLatency)/1e6)
}
```

## Dual-Stack Comparison Test

```go
package main

import (
    "fmt"
    "net"
    "net/http"
    "time"
)

type ProtocolResult struct {
    Protocol string
    Latency  time.Duration
    Status   int
    Error    error
}

func testBothProtocols(host string, port int) [2]ProtocolResult {
    var results [2]ProtocolResult
    done := make(chan ProtocolResult, 2)

    // Test IPv6
    go func() {
        start := time.Now()
        addr := fmt.Sprintf("[%s]:%d", host, port)
        conn, err := net.DialTimeout("tcp6", addr, 5*time.Second)
        latency := time.Since(start)
        if err != nil {
            done <- ProtocolResult{"IPv6", latency, 0, err}
            return
        }
        conn.Close()
        done <- ProtocolResult{"IPv6", latency, 200, nil}
    }()

    // Test IPv4 (if dual-stack DNS)
    go func() {
        time.Sleep(50 * time.Millisecond)  // Happy Eyeballs delay
        start := time.Now()
        // Resolve IPv4 for same hostname
        addrs, err := net.LookupHost(host)
        if err != nil {
            done <- ProtocolResult{"IPv4", 0, 0, err}
            return
        }
        var ipv4 string
        for _, a := range addrs {
            if net.ParseIP(a).To4() != nil {
                ipv4 = a
                break
            }
        }
        if ipv4 == "" {
            done <- ProtocolResult{"IPv4", 0, 0, fmt.Errorf("no IPv4 address")}
            return
        }
        conn, err := net.DialTimeout("tcp4", fmt.Sprintf("%s:%d", ipv4, port), 5*time.Second)
        latency := time.Since(start)
        if err != nil {
            done <- ProtocolResult{"IPv4", latency, 0, err}
            return
        }
        conn.Close()
        done <- ProtocolResult{"IPv4", latency, 200, nil}
    }()

    for i := 0; i < 2; i++ {
        r := <-done
        results[i] = r
    }
    return results
}

func main() {
    hosts := []struct{ host string; port int }{
        {"2001:db8::server", 80},
        {"2001:db8::server", 443},
    }
    for _, h := range hosts {
        fmt.Printf("\nTesting %s:%d\n", h.host, h.port)
        results := testBothProtocols(h.host, h.port)
        for _, r := range results {
            if r.Error != nil {
                fmt.Printf("  %-6s: ERROR %v\n", r.Protocol, r.Error)
            } else {
                fmt.Printf("  %-6s: %.2f ms\n", r.Protocol, float64(r.Latency.Microseconds())/1000)
            }
        }
    }
}
```

## TCP Connection Flood Test (Rate-Limited)

```go
package main

import (
    "fmt"
    "net"
    "sync"
    "sync/atomic"
    "time"
)

func tcpConnectionTest(target string, ratePerSec int, duration time.Duration) {
    var established int64
    var failed int64

    interval := time.Second / time.Duration(ratePerSec)
    end := time.Now().Add(duration)
    var wg sync.WaitGroup

    fmt.Printf("TCP connection test: %s at %d conn/s for %v\n",
        target, ratePerSec, duration)

    for time.Now().Before(end) {
        wg.Add(1)
        go func() {
            defer wg.Done()
            conn, err := net.DialTimeout("tcp6", target, 5*time.Second)
            if err != nil {
                atomic.AddInt64(&failed, 1)
                return
            }
            atomic.AddInt64(&established, 1)
            conn.Close()
        }()
        time.Sleep(interval)
    }
    wg.Wait()

    total := established + failed
    fmt.Printf("Results: %d total, %d established (%.1f%%), %d failed\n",
        total, established, float64(established)/float64(total)*100, failed)
}

func main() {
    tcpConnectionTest("[2001:db8::server]:80", 100, 30*time.Second)
}
```

## Conclusion

IPv6 load testing in Go uses `net.DialTimeout("tcp6", "[addr]:port", timeout)` to force IPv6 connections and `http.Transport` with a custom `DialContext` to control address family selection. Use `sync/atomic` for thread-safe counters and goroutine pools with a semaphore channel to control concurrency. Compare IPv6 vs IPv4 performance by racing both protocols — a latency gap greater than 5ms indicates routing or peering issues. For production load testing, use existing tools like `wrk` or `k6` with IPv6 targets (`http://[2001:db8::1]/`); build custom Go load testers when you need IPv6-specific metrics or protocol-level testing of TCP connection establishment rates.
