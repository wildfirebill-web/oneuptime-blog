# How to Set Up gRPC Service Discovery Using IPv4 DNS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, DNS, Service Discovery, IPv4, Go, Kubernetes

Description: Learn how to configure gRPC clients to use DNS-based service discovery with IPv4 A records, enabling automatic load balancing across multiple server instances in Kubernetes and Docker environments.

## How gRPC DNS Discovery Works

```text
gRPC Client
    │
    ▼
DNS Resolver (dns:///)
    │  resolves "service-name:port" → multiple A records
    ▼
┌──────────────────────────────┐
│ A record 10.244.1.5:50051    │
│ A record 10.244.2.8:50051    │  ← all endpoints
│ A record 10.244.3.12:50051   │
└──────────────────────────────┘
    │  round_robin policy picks endpoint
    ▼
gRPC Server Pod
```

## Go: DNS Resolver with Round-Robin

```go
package main

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    pb "example.com/hello"
)

func main() {
    // "dns:///" prefix activates gRPC's built-in DNS resolver
    // The resolver looks up A records and connects to all of them
    conn, err := grpc.NewClient(
        "dns:///greeter.default.svc.cluster.local:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithDefaultServiceConfig(`{
            "loadBalancingPolicy": "round_robin",
            "methodConfig": [{
                "name": [{}],
                "waitForReady": true,
                "timeout": "3s",
                "retryPolicy": {
                    "maxAttempts": 3,
                    "initialBackoff": "0.1s",
                    "maxBackoff": "1s",
                    "backoffMultiplier": 2,
                    "retryableStatusCodes": ["UNAVAILABLE"]
                }
            }]
        }`),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    client := pb.NewGreeterClient(conn)
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "world"})
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("Response: %s", resp.GetMessage())
}
```

## Python: DNS-Based Discovery

```python
import grpc
import hello_pb2
import hello_pb2_grpc

# dns:/// triggers the gRPC built-in DNS resolver

channel = grpc.insecure_channel(
    "dns:///greeter.default.svc.cluster.local:50051",
    options=[
        ("grpc.lb_policy_name",           "round_robin"),
        ("grpc.service_config", '{"loadBalancingPolicy":"round_robin"}'),
    ],
)

stub = hello_pb2_grpc.GreeterStub(channel)
resp = stub.SayHello(hello_pb2.HelloRequest(name="world"), timeout=3.0)
print(resp.message)
```

## Kubernetes: Headless Service Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: greeter
  namespace: default
spec:
  clusterIP: None   # Headless - DNS returns all matching pod IPs
  selector:
    app: greeter
  ports:
    - name: grpc
      port: 50051
      targetPort: 50051
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: greeter
spec:
  replicas: 3
  selector:
    matchLabels:
      app: greeter
  template:
    metadata:
      labels:
        app: greeter
    spec:
      containers:
        - name: greeter
          image: myrepo/greeter:latest
          ports:
            - containerPort: 50051
```

## Verify DNS Resolution

```bash
# From inside a pod
nslookup greeter.default.svc.cluster.local
# Should return one A record per pod

# With dig
dig greeter.default.svc.cluster.local A
```

## Conclusion

Use the `dns:///` scheme prefix in the gRPC target to enable DNS-based discovery. Combine with the `round_robin` load balancing policy to distribute RPCs across all resolved addresses. In Kubernetes, a headless Service (no `clusterIP`) returns individual pod IPs from DNS, enabling true client-side load balancing. Add `waitForReady: true` in the service config to make the channel wait for DNS to resolve instead of failing immediately if the service is not yet available.
