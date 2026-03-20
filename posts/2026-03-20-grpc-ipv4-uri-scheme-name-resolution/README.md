# How to Use the ipv4 URI Scheme in gRPC Name Resolution

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, IPv4, Name Resolution, Go, Python, Networking

Description: Use the ipv4:// URI scheme in gRPC to create direct IPv4 connections, configure custom name resolvers, and handle multi-address load balancing in Go and Python.

## Introduction

gRPC supports pluggable name resolution. The built-in `ipv4` scheme allows you to specify one or more IPv4 endpoints directly without DNS lookup, which is useful in environments where DNS is unavailable or unreliable.

## Basic ipv4:// Target in Go

```go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    pb "example.com/proto/helloworld"
)

func main() {
    // Single IPv4 target
    conn, err := grpc.Dial(
        "ipv4:///192.168.1.10:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    client := pb.NewGreeterClient(conn)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "world"})
    if err != nil {
        log.Fatal(err)
    }
    log.Println(resp.Message)
}
```

## Multiple IPv4 Addresses with Round-Robin

```go
import (
    "google.golang.org/grpc/balancer/roundrobin"
    _ "google.golang.org/grpc/balancer/roundrobin" // register
)

conn, err := grpc.Dial(
    "ipv4:///192.168.1.10:50051,192.168.1.11:50051,192.168.1.12:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
)
```

## Python gRPC with ipv4 Target

```python
import grpc
import helloworld_pb2
import helloworld_pb2_grpc

channel = grpc.insecure_channel(
    "ipv4:///192.168.1.10:50051",
    options=[
        ("grpc.lb_policy_name", "round_robin"),
        ("grpc.dns_min_time_between_resolutions_ms", 30000),
    ]
)

stub = helloworld_pb2_grpc.GreeterStub(channel)
response = stub.SayHello(helloworld_pb2.HelloRequest(name="World"))
print(response.message)
```

## Python - Multiple Addresses

```python
channel = grpc.insecure_channel(
    "ipv4:///192.168.1.10:50051,192.168.1.11:50051",
    options=[("grpc.lb_policy_name", "round_robin")]
)
```

## Custom Static Resolver in Go

```go
package main

import (
    "google.golang.org/grpc/resolver"
)

type staticResolver struct {
    cc resolver.ClientConn
}

func (r *staticResolver) ResolveNow(resolver.ResolveNowOptions) {}
func (r *staticResolver) Close() {}

type staticResolverBuilder struct{}

func (b *staticResolverBuilder) Build(target resolver.Target,
    cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {

    addrs := []resolver.Address{
        {Addr: "192.168.1.10:50051"},
        {Addr: "192.168.1.11:50051"},
    }
    cc.UpdateState(resolver.State{Addresses: addrs})
    return &staticResolver{cc: cc}, nil
}

func (b *staticResolverBuilder) Scheme() string { return "static" }

func init() {
    resolver.Register(&staticResolverBuilder{})
}
```

## Service Config with Health Checking

```go
serviceConfig := `{
    "loadBalancingPolicy": "round_robin",
    "healthCheckConfig": {"serviceName": ""}
}`

conn, _ := grpc.Dial(
    "ipv4:///192.168.1.10:50051",
    grpc.WithDefaultServiceConfig(serviceConfig),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

## Conclusion

The `ipv4://` scheme bypasses DNS and connects directly to specified IPv4 addresses. Combine it with `round_robin` load balancing policy to distribute calls across multiple backends. For dynamic address sets, implement a custom `resolver.Builder` that pushes updated address lists to the gRPC channel.
