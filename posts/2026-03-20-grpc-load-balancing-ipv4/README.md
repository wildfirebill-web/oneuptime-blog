# How to Implement gRPC Load Balancing with IPv4 Endpoints

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, Load Balancing, IPv4, Python, Go, Kubernetes

Description: Learn how to implement client-side and server-side load balancing for gRPC services using IPv4 endpoints, including round-robin, DNS-based discovery, and Nginx/Envoy proxy patterns.

## Client-Side Round-Robin (Go)

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/balancer/roundrobin"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/resolver"
    pb "example.com/hello"
)

func main() {
    // Static IP list resolver for demo
    // In production use DNS resolver with multiple A records
    conn, err := grpc.NewClient(
        "dns:///greeter.default.svc.cluster.local:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    client := pb.NewGreeterClient(conn)
    for i := 0; i < 5; i++ {
        ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
        resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "world"})
        cancel()
        if err != nil {
            log.Printf("RPC failed: %v", err)
            continue
        }
        fmt.Printf("Response: %s\n", resp.GetMessage())
    }
}
```

## Python: Client-Side Round-Robin

```python
import grpc
import hello_pb2
import hello_pb2_grpc

# DNS-based round-robin: resolve all A records and balance across them
channel = grpc.insecure_channel(
    "dns:///greeter.default.svc.cluster.local:50051",
    options=[("grpc.lb_policy_name", "round_robin")],
)

stub = hello_pb2_grpc.GreeterStub(channel)
for _ in range(5):
    try:
        resp = stub.SayHello(hello_pb2.HelloRequest(name="world"), timeout=3.0)
        print(resp.message)
    except grpc.RpcError as e:
        print(f"Error: {e.code()}")
```

## Nginx: Server-Side gRPC Load Balancer

```nginx
# Nginx 1.13.10+ supports gRPC proxying natively
upstream grpc_backends {
    least_conn;
    server 10.0.0.1:50051;
    server 10.0.0.2:50051;
    server 10.0.0.3:50051;
    keepalive 32;
}

server {
    listen 50051 http2;

    location / {
        grpc_pass grpc://grpc_backends;
        error_page 502 = /error502grpc;
    }

    location = /error502grpc {
        internal;
        default_type application/grpc;
        add_header grpc-status 14;  # UNAVAILABLE
        add_header content-length 0;
        return 204;
    }
}
```

## Kubernetes: Headless Service for Client-Side LB

```yaml
# Headless service — DNS returns all pod IPs
apiVersion: v1
kind: Service
metadata:
  name: greeter
spec:
  clusterIP: None   # headless
  selector:
    app: greeter
  ports:
    - port: 50051
```

```
# DNS A record resolution
nslookup greeter.default.svc.cluster.local
→ 10.244.1.5
→ 10.244.2.8
→ 10.244.3.12
```

The gRPC client resolves all A records and round-robins across them with `round_robin` policy.

## Conclusion

gRPC load balancing works at the application layer (HTTP/2 multiplexing), which is invisible to TCP-level load balancers. Use the `round_robin` load balancing policy with DNS resolution to distribute across all pod IPs from a Kubernetes headless Service. For server-side load balancing, configure Nginx with `grpc_pass` or use Envoy, which is the standard data plane in service meshes. Avoid ClusterIP services for gRPC unless you use a Layer-7 proxy, because TCP connections are long-lived and a single connection will always land on the same pod.
