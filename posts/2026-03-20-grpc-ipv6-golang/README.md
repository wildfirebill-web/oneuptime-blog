# How to Configure gRPC Servers with IPv6 in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, Go, IPv6, Networking, API

Description: Configure gRPC servers and clients in Go to listen on and connect via IPv6 addresses for dual-stack or IPv6-only deployments.

## Overview

Go's `net` package and the `google.golang.org/grpc` library fully support IPv6. The key is using the correct address format when creating listeners and clients.

## Step 1: gRPC Server Listening on IPv6

```go
package main

import (
    "fmt"
    "log"
    "net"

    "google.golang.org/grpc"
    pb "example.com/helloworld"
)

type server struct {
    pb.UnimplementedGreeterServer
}

func main() {
    // Listen on all IPv6 interfaces (:: is IPv6 equivalent of 0.0.0.0)
    // Use [::]:50051 format — square brackets are required for IPv6
    listener, err := net.Listen("tcp", "[::]:50051")
    if err != nil {
        log.Fatalf("Failed to listen: %v", err)
    }

    s := grpc.NewServer()
    pb.RegisterGreeterServer(s, &server{})

    fmt.Printf("gRPC server listening on %s\n", listener.Addr().String())

    if err := s.Serve(listener); err != nil {
        log.Fatalf("Failed to serve: %v", err)
    }
}
```

## Step 2: Listen on Both IPv4 and IPv6 (Dual-Stack)

On Linux, `[::]` typically accepts both IPv4 and IPv6 unless `net.ipv6only` is set:

```go
// Check if dual-stack is working
listener, err := net.Listen("tcp6", "[::]:50051")
// tcp6 forces IPv6-only; use "tcp" for dual-stack

// For explicit dual-stack, run two listeners
go func() {
    ln4, _ := net.Listen("tcp4", "0.0.0.0:50051")
    s4 := grpc.NewServer()
    pb.RegisterGreeterServer(s4, &server{})
    s4.Serve(ln4)
}()

ln6, _ := net.Listen("tcp6", "[::]:50051")
s6 := grpc.NewServer()
pb.RegisterGreeterServer(s6, &server{})
s6.Serve(ln6)
```

## Step 3: gRPC Client Connecting to IPv6

```go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    pb "example.com/helloworld"
)

func main() {
    // Connect to IPv6 gRPC server
    // Square brackets are required around IPv6 addresses
    target := "[2001:db8::1]:50051"

    conn, err := grpc.NewClient(
        target,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewGreeterClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "World"})
    if err != nil {
        log.Fatalf("Could not greet: %v", err)
    }
    log.Printf("Greeting: %s", resp.GetMessage())
}
```

## Step 4: gRPC with TLS on IPv6

```go
import (
    "crypto/tls"
    "google.golang.org/grpc/credentials"
)

// Server with TLS on IPv6
func startTLSServer() {
    // Load certificate
    cert, err := tls.LoadX509KeyPair("server.crt", "server.key")
    if err != nil {
        log.Fatal(err)
    }

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        MinVersion:   tls.VersionTLS12,
    }

    creds := credentials.NewTLS(tlsConfig)
    s := grpc.NewServer(grpc.Creds(creds))

    // Listen on IPv6 with TLS
    ln, _ := net.Listen("tcp", "[::]:443")
    pb.RegisterGreeterServer(s, &server{})
    s.Serve(ln)
}

// Client connecting to IPv6 gRPC with TLS
func connectWithTLS() {
    creds, _ := credentials.NewClientTLSFromFile("ca.crt", "example.com")

    conn, _ := grpc.NewClient(
        "[2001:db8::1]:443",
        grpc.WithTransportCredentials(creds),
    )
    defer conn.Close()
}
```

## Step 5: IPv6-Aware Health Check Server

```go
import (
    "google.golang.org/grpc/health"
    "google.golang.org/grpc/health/grpc_health_v1"
)

// Add health check service to your gRPC server
healthServer := health.NewServer()
grpc_health_v1.RegisterHealthServer(s, healthServer)

// Set service health status
healthServer.SetServingStatus("greeter", grpc_health_v1.HealthCheckResponse_SERVING)
```

## Testing

```bash
# Test gRPC server over IPv6 with grpcurl
grpcurl -plaintext '[2001:db8::1]:50051' list
grpcurl -plaintext '[2001:db8::1]:50051' helloworld.Greeter/SayHello

# Test health check
grpcurl -plaintext '[2001:db8::1]:50051' grpc.health.v1.Health/Check
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to monitor your gRPC health endpoints over IPv6. Configure TCP port monitors for port 50051 on your IPv6 address and set up custom HTTP monitors for the gRPC health check protocol.

## Conclusion

Go gRPC servers listen on IPv6 by using `[::]` in the listen address. Clients connect using `[ipv6addr]:port` format with square brackets. All standard gRPC features including TLS, health checks, and interceptors work identically over IPv6.
