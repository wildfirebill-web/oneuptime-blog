# How to Use Dapr with WebAssembly Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, WebAssembly, WASM, Component, Middleware

Description: Learn how to use WebAssembly middleware components with Dapr to run portable, sandboxed business logic in the sidecar pipeline without a separate service.

---

Dapr supports WebAssembly (WASM) as a middleware component type, allowing you to compile portable logic into `.wasm` binaries and run them directly in the Dapr sidecar pipeline. This enables lightweight, sandboxed transformations without deploying additional services.

## How Dapr WASM Middleware Works

WASM middleware runs in the Dapr sidecar's HTTP middleware pipeline. When a request passes through the sidecar (inbound or outbound), the compiled WASM module can inspect and modify the request or response.

```text
Client Request
     |
     v
Dapr Sidecar HTTP Pipeline
  [WASM Middleware] <-- your .wasm module runs here
     |
     v
Your Application
```

## Writing a WASM Middleware Module

Dapr uses the [http-wasm](https://http-wasm.io/) ABI. Write middleware in Go using the `http-wasm/handler` package:

```go
package main

import (
    "github.com/http-wasm/http-wasm-guest-tinygo/handler"
    "github.com/http-wasm/http-wasm-guest-tinygo/handler/api"
)

func main() {
    handler.HandleRequestFn = handleRequest
}

func handleRequest(req api.Request, resp api.Response) (next bool, reqCtx uint32) {
    // Add a custom header to every inbound request
    req.Headers().Set("x-dapr-wasm", "processed")

    // Log the request path
    _ = req.GetURI()

    return true, 0  // continue to next handler
}
```

Compile to WASM using TinyGo:

```bash
tinygo build -o my-middleware.wasm -scheduler=none -target=wasi ./main.go
```

## Configuring the WASM Component

Create a Dapr HTTP middleware component pointing to the compiled binary:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: wasm-middleware
spec:
  type: middleware.http.wasm
  version: v1
  metadata:
    - name: url
      value: "file:///dapr/middleware/my-middleware.wasm"
    - name: guestConfig
      value: '{"logLevel": "info"}'
```

For Kubernetes, mount the `.wasm` file via a ConfigMap or init container:

```yaml
initContainers:
  - name: copy-wasm
    image: busybox
    command: ["sh", "-c", "cp /wasm-source/my-middleware.wasm /dapr/middleware/"]
    volumeMounts:
      - name: wasm-volume
        mountPath: /dapr/middleware
```

## Configuring the Middleware Pipeline

Reference the middleware in a Dapr Configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  httpPipeline:
    handlers:
      - name: wasm-middleware
        type: middleware.http.wasm
```

Apply it to your deployment:

```yaml
annotations:
  dapr.io/config: "appconfig"
```

## Use Cases for WASM Middleware

- Request header validation or enrichment
- Rate limiting based on custom business rules
- Request/response payload transformation
- A/B testing traffic routing logic
- Lightweight authentication token inspection

## Summary

Dapr WASM middleware lets you compile portable Go (or Rust/C) logic into `.wasm` binaries that run directly in the Dapr sidecar HTTP pipeline. Configure a `middleware.http.wasm` component pointing to the compiled binary, reference it in a Dapr Configuration, and the middleware executes for every request passing through the sidecar without deploying additional services.
