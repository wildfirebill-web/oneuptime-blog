# How to Use Dapr Kitex Output Binding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, Kitex, RPC, Go

Description: Learn how to configure the Dapr Kitex output binding to call CloudWeGo Kitex RPC services from Dapr-enabled applications in a Go microservices environment.

---

## Overview of the Dapr Kitex Binding

Kitex is a high-performance Go RPC framework developed by ByteDance and open-sourced as part of the CloudWeGo project. The Dapr Kitex output binding enables Dapr applications to invoke Kitex services using Thrift serialization without writing Kitex client code directly.

## Prerequisites

A running Kitex server registered with a service registry (ZooKeeper or Etcd). This example uses ZooKeeper.

## Configure the Kitex Output Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kitex-service
spec:
  type: bindings.kitex
  version: v1
  metadata:
  - name: hostPorts
    value: "localhost:8888"
  - name: destService
    value: "order.OrderService"
  - name: version
    value: "v1"
```

## Invoke a Kitex Service Method

```bash
curl -X POST http://localhost:3500/v1.0/bindings/kitex-service \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "invoke",
    "data": {
      "orderId": "order-001",
      "customerId": "cust-123"
    },
    "metadata": {
      "methodName": "GetOrder",
      "serviceName": "order.OrderService",
      "version": "v1",
      "headersKey": "traceId",
      "headersValue": "trace-abc-123"
    }
  }'
```

## Calling from Application Code

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type BindingRequest struct {
    Operation string            `json:"operation"`
    Data      interface{}       `json:"data"`
    Metadata  map[string]string `json:"metadata"`
}

func callKitexService(method string, data interface{}) ([]byte, error) {
    req := BindingRequest{
        Operation: "invoke",
        Data:      data,
        Metadata: map[string]string{
            "methodName":  method,
            "serviceName": "order.OrderService",
            "version":     "v1",
        },
    }

    body, _ := json.Marshal(req)
    resp, err := http.Post(
        "http://localhost:3500/v1.0/bindings/kitex-service",
        "application/json",
        bytes.NewReader(body),
    )
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result []byte
    json.NewDecoder(resp.Body).Decode(&result)
    return result, nil
}

func main() {
    result, err := callKitexService("GetOrder", map[string]string{
        "orderId": "order-001",
    })
    if err != nil {
        panic(err)
    }
    fmt.Println("Result:", string(result))
}
```

## Passing Custom Headers

Kitex supports passing metadata through RPC headers for tracing and context:

```bash
curl -X POST http://localhost:3500/v1.0/bindings/kitex-service \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "invoke",
    "data": {"userId": "u-456"},
    "metadata": {
      "methodName": "GetUser",
      "serviceName": "user.UserService",
      "headersKey": "x-request-id,x-tenant-id",
      "headersValue": "req-abc-123,tenant-42"
    }
  }'
```

Multiple headers use comma-separated keys and values in matching order.

## Direct Connection vs Service Discovery

For development, use direct `hostPorts` connection:

```yaml
metadata:
- name: hostPorts
  value: "localhost:8888"
```

For production with service discovery:

```yaml
metadata:
- name: destService
  value: "order.OrderService"
- name: registryAddress
  value: "zookeeper://zk:2181"
```

## Summary

The Dapr Kitex output binding enables Dapr applications to call CloudWeGo Kitex RPC services without writing Kitex client code. Configure the destination service and host in the component YAML, then invoke methods by specifying the service name, method name, and data payload. Custom headers support tracing and multi-tenant contexts across RPC calls.
