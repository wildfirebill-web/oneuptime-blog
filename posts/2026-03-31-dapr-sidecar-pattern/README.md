# How to Implement Sidecar Pattern with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sidecar, Pattern, Kubernetes, Architecture

Description: Learn how Dapr implements the sidecar pattern to offload cross-cutting concerns like state, pub/sub, and tracing from your application code.

---

## Overview

The sidecar pattern deploys a helper process alongside the main application container in the same pod. The sidecar handles cross-cutting concerns - Dapr's sidecar (daprd) provides state management, pub/sub, service invocation, secrets, and more without modifying application code.

## How Dapr's Sidecar Works

```text
Pod
+-----------------------------+
|  App Container (port 8080)  |
|  Dapr Sidecar (port 3500)   |
+-----------------------------+
         |
    Dapr Control Plane (Kubernetes)
```

When your app needs to call another service or access state, it calls the Dapr sidecar on localhost:3500. The sidecar handles service discovery, mTLS, retries, and tracing transparently.

## Enabling the Sidecar in Kubernetes

Annotate your pod spec to inject the Dapr sidecar:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
        dapr.io/config: "daprconfig"
        dapr.io/log-level: "info"
        dapr.io/sidecar-cpu-request: "100m"
        dapr.io/sidecar-memory-request: "64Mi"
        dapr.io/sidecar-cpu-limit: "500m"
        dapr.io/sidecar-memory-limit: "256Mi"
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.0
          ports:
            - containerPort: 8080
```

## Using the Sidecar HTTP API

Your application talks to the sidecar via localhost:

```go
package main

import (
    "fmt"
    "net/http"
    "strings"
)

const daprPort = 3500

// Save state via sidecar
func saveState(key, value string) error {
    body := fmt.Sprintf(`[{"key":"%s","value":"%s"}]`, key, value)
    resp, err := http.Post(
        fmt.Sprintf("http://localhost:%d/v1.0/state/statestore", daprPort),
        "application/json",
        strings.NewReader(body),
    )
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    return nil
}

// Invoke another service via sidecar
func invokeService(appID, method string) ([]byte, error) {
    resp, err := http.Get(
        fmt.Sprintf("http://localhost:%d/v1.0/invoke/%s/method/%s", daprPort, appID, method),
    )
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    // read response...
    return nil, nil
}
```

## Using the Dapr SDK (Recommended)

```go
package main

import (
    "context"
    dapr "github.com/dapr/go-sdk/client"
)

func main() {
    // SDK communicates with local sidecar automatically
    client, err := dapr.NewClient()
    if err != nil {
        panic(err)
    }
    defer client.Close()

    ctx := context.Background()

    // State management
    err = client.SaveState(ctx, "statestore", "session:user1", []byte(`{"cart":["item1","item2"]}`), nil)

    // Service invocation
    data, err := client.InvokeMethod(ctx, "product-service", "/products/123", "GET")

    // Pub/Sub
    err = client.PublishEvent(ctx, "pubsub", "order-placed", map[string]string{"orderId": "ord-001"})
}
```

## Sidecar Health Check

The sidecar exposes health endpoints your readiness probes can use:

```yaml
livenessProbe:
  httpGet:
    path: /v1.0/healthz
    port: 3500
  initialDelaySeconds: 5
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /v1.0/healthz/outbound
    port: 3500
  initialDelaySeconds: 5
  periodSeconds: 10
```

## Running Locally Without Kubernetes

```bash
dapr run --app-id order-service --app-port 8080 --dapr-http-port 3500 -- go run main.go
```

The `dapr run` command starts the sidecar process alongside your application locally.

## Summary

Dapr's sidecar pattern cleanly separates application business logic from infrastructure concerns. By injecting the daprd sidecar into your pod, your service gains state, pub/sub, service invocation, secrets, and observability capabilities through a simple localhost HTTP or gRPC API. The sidecar handles all protocol translation, service discovery, and security transparently.
