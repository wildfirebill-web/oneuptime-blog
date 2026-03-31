# How to Develop Dapr Pluggable State Store Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pluggable Component, State Store, gRPC, Extension

Description: Build a custom Dapr pluggable state store component using the gRPC-based component SDK to connect Dapr to any backing store not supported out of the box.

---

## What Are Pluggable Components?

Dapr's built-in components cover dozens of backing services, but organizations often have internal databases or custom storage systems. Pluggable components let you implement a Dapr component interface and run it as a separate process or container - Dapr communicates with it over a Unix domain socket using gRPC.

## Setting Up the Component SDK

```bash
# Initialize a Go project for the pluggable component
mkdir dapr-custom-statestore && cd dapr-custom-statestore
go mod init github.com/myorg/dapr-custom-statestore

# Install the Dapr pluggable component SDK
go get github.com/dapr-sandbox/components-go-sdk@latest
```

## Implementing the State Store Interface

The state store interface requires implementing Init, Features, Get, Set, Delete, and Ping:

```go
package main

import (
    "context"
    "github.com/dapr-sandbox/components-go-sdk/state/v1"
    proto "github.com/dapr/dapr/pkg/proto/components/v1"
)

type MyCustomStore struct {
    storage map[string][]byte
}

func (s *MyCustomStore) Init(ctx context.Context, req *proto.InitRequest) (*proto.InitResponse, error) {
    // Initialize connection to your backing store
    s.storage = make(map[string][]byte)
    return &proto.InitResponse{}, nil
}

func (s *MyCustomStore) Features(ctx context.Context, req *proto.FeaturesRequest) (*proto.FeaturesResponse, error) {
    return &proto.FeaturesResponse{
        Features: []string{"ETAG", "TRANSACTIONAL"},
    }, nil
}

func (s *MyCustomStore) Get(ctx context.Context, req *proto.GetRequest) (*proto.GetResponse, error) {
    val, ok := s.storage[req.Key]
    if !ok {
        return &proto.GetResponse{}, nil
    }
    return &proto.GetResponse{Data: val}, nil
}

func (s *MyCustomStore) Set(ctx context.Context, req *proto.SetRequest) (*proto.SetResponse, error) {
    s.storage[req.Key] = req.Value
    return &proto.SetResponse{}, nil
}

func (s *MyCustomStore) Delete(ctx context.Context, req *proto.DeleteRequest) (*proto.DeleteResponse, error) {
    delete(s.storage, req.Key)
    return &proto.DeleteResponse{}, nil
}

func (s *MyCustomStore) Ping(ctx context.Context, req *proto.PingRequest) (*proto.PingResponse, error) {
    return &proto.PingResponse{}, nil
}
```

## Registering and Running the Component

```go
package main

import (
    dapr "github.com/dapr-sandbox/components-go-sdk"
    state "github.com/dapr-sandbox/components-go-sdk/state/v1"
)

func main() {
    dapr.Register("my-custom-statestore", dapr.WithStateStore(func() state.Store {
        return &MyCustomStore{}
    }))

    dapr.MustRun()
}
```

## Creating the Component Manifest

Reference the pluggable component in a Dapr component YAML:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: customstore
spec:
  type: state.my-custom-statestore
  version: v1
  metadata:
    - name: connectionString
      value: "custom-db://localhost:5432/mydb"
```

## Running the Pluggable Component Locally

```bash
# Build the component
go build -o custom-statestore .

# Run it - it creates a Unix socket for Dapr to connect to
DAPR_COMPONENT_SOCKET_FOLDER=/tmp/dapr-components \
  ./custom-statestore &

# Run Dapr with the component socket path
dapr run \
  --app-id my-app \
  --app-port 8080 \
  --components-path ./components \
  --unix-domain-socket /tmp/dapr-components \
  -- ./my-app
```

## Transactional Support

For transactional state stores, implement the Transact method:

```go
func (s *MyCustomStore) Transact(ctx context.Context, req *proto.TransactionalStateRequest) (*proto.TransactionalStateResponse, error) {
    // Begin transaction
    for _, op := range req.Operations {
        switch op.Request.(type) {
        case *proto.TransactionalStateOperation_Set:
            setOp := op.GetSet()
            s.storage[setOp.Key] = setOp.Value
        case *proto.TransactionalStateOperation_Delete:
            deleteOp := op.GetDelete()
            delete(s.storage, deleteOp.Key)
        }
    }
    return &proto.TransactionalStateResponse{}, nil
}
```

## Summary

Dapr pluggable state store components let you connect Dapr to any backing store by implementing a gRPC-based interface. The components-go-sdk reduces boilerplate, and the Unix domain socket transport ensures low-latency communication between Dapr and your component process. This extensibility makes it possible to adopt Dapr without being limited to its built-in component catalog.
