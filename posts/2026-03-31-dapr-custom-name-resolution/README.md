# How to Use Custom Name Resolution in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Name Resolution, Custom Component, Plugin, Service Discovery

Description: Learn how to build and register a custom name resolution component in Dapr using the pluggable component SDK for specialized service discovery needs.

---

## When to Build a Custom Name Resolution Component

Dapr ships with built-in name resolution components for Kubernetes DNS, mDNS, Consul, SQLite, and NameFormat. However, you may need a custom implementation for:

- Integration with proprietary service registries
- Custom load balancing or affinity logic
- Special routing based on metadata or request context
- A registry not supported by built-in components (e.g., Zookeeper, etcd)

Dapr supports pluggable components via gRPC, allowing you to implement custom name resolution in any language.

## Implementing a Custom Name Resolution Component

Define the gRPC service by implementing the Dapr pluggable component interface. Using Go:

```go
package main

import (
    "context"
    "net"

    proto "github.com/dapr/dapr/pkg/proto/components/v1"
    "google.golang.org/grpc"
)

type MyNameResolver struct {
    proto.UnimplementedNameResolutionServer
    registry map[string]string
}

func (r *MyNameResolver) InitWithMetadata(
    ctx context.Context,
    req *proto.InitRequest,
) (*proto.Empty, error) {
    r.registry = map[string]string{
        "order-service":   "10.0.1.10:50001",
        "payment-service": "10.0.1.11:50001",
    }
    return &proto.Empty{}, nil
}

func (r *MyNameResolver) ResolveID(
    ctx context.Context,
    req *proto.ResolveRequest,
) (*proto.ResolveResponse, error) {
    addr, ok := r.registry[req.Id]
    if !ok {
        return nil, fmt.Errorf("app ID %s not found", req.Id)
    }
    return &proto.ResolveResponse{Address: addr}, nil
}

func main() {
    listener, _ := net.Listen("unix", "/tmp/dapr-my-resolver.sock")
    server := grpc.NewServer()
    proto.RegisterNameResolutionServer(server, &MyNameResolver{})
    server.Serve(listener)
}
```

Build the binary:

```bash
go build -o my-resolver ./cmd/resolver
```

## Registering the Custom Component

Create the component YAML pointing to the socket file:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nameresolution
spec:
  type: nameresolution.my-resolver
  version: v1
  metadata:
    - name: registryEndpoint
      value: "etcd://etcd.internal:2379"
```

Place the socket file in the Dapr components Unix socket directory (default `/tmp/dapr-components-sockets/`).

## Running the Custom Resolver with Dapr

Start the custom resolver binary before starting Dapr:

```bash
./my-resolver &

dapr run --app-id myapp \
  --components-path ./components \
  -- ./myapp
```

Dapr detects the socket file and loads the custom component at startup.

## Packaging in Kubernetes

In Kubernetes, run the custom resolver as a sidecar container alongside the Dapr sidecar:

```yaml
spec:
  containers:
    - name: app
      image: myapp:latest
    - name: my-resolver
      image: myrepo/my-resolver:latest
      volumeMounts:
        - name: dapr-sockets
          mountPath: /tmp/dapr-components-sockets
  volumes:
    - name: dapr-sockets
      emptyDir: {}
```

## Testing the Custom Resolver

Invoke a service to verify the custom resolver is handling requests:

```bash
curl http://localhost:3500/v1.0/invoke/order-service/method/health
```

Check that the address returned by your resolver is the one being connected to:

```bash
dapr run --app-id myapp --log-level debug \
  --components-path ./components -- ./myapp 2>&1 | grep -i resolve
```

## Summary

Custom Dapr name resolution components are implemented as gRPC services using the pluggable component interface. Expose the resolver via a Unix socket, register it with a component YAML, and run it as a sidecar in Kubernetes. This approach enables integration with any service registry while preserving Dapr's standard service invocation API.
