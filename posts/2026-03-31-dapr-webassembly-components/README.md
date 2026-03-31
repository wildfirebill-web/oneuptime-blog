# How to Use Dapr with WebAssembly Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, WebAssembly, WASM, Component, Extension

Description: Learn how to use Dapr's WebAssembly middleware and component support to run portable, sandboxed business logic inside the Dapr sidecar using WASM modules.

---

## Dapr and WebAssembly

Dapr supports WebAssembly (WASM) as a middleware execution environment. WASM modules run inside the Dapr sidecar, providing a sandboxed, language-agnostic way to implement custom middleware for HTTP pipeline processing without deploying additional services.

## Use Cases for WASM in Dapr

- **Request transformation** - Modify headers or body before forwarding
- **Custom authentication** - Validate tokens using portable WASM logic
- **Rate limiting** - Implement custom rate limit algorithms
- **Data validation** - Schema validation before state writes

## Writing a WASM Middleware Module in Go

Use the TinyGo compiler to produce WASM:

```go
// main.go - Compile with TinyGo to WASM
package main

import "unsafe"

//export handle_request
func handleRequest(ptr, size uint32) uint64 {
    // Read request bytes from shared memory
    body := ptrToString(ptr, size)
    _ = body

    // Transform: add a custom header indicator
    result := `{"processed": true}`
    return stringToPtr(result)
}

func ptrToString(ptr, size uint32) string {
    return unsafe.String((*byte)(unsafe.Pointer(uintptr(ptr))), size)
}

func stringToPtr(s string) uint64 {
    b := []byte(s)
    ptr := &b[0]
    unsafePtr := uintptr(unsafe.Pointer(ptr))
    return (uint64(unsafePtr) << uint64(32)) | uint64(len(b))
}

func main() {}
```

Compile to WASM:

```bash
tinygo build -o middleware.wasm -target wasi ./main.go
```

## Configuring Dapr WASM Middleware

Create a middleware component that references your WASM file:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: wasm-transformer
spec:
  type: middleware.http.wasm
  version: v1
  metadata:
    - name: url
      value: "file:///components/middleware.wasm"
```

## Applying the Middleware in a Pipeline

Chain the WASM middleware into the Dapr HTTP pipeline via Configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: pipeline-config
spec:
  httpPipeline:
    handlers:
      - name: wasm-transformer
        type: middleware.http.wasm
```

Annotate your deployment to use this configuration:

```yaml
annotations:
  dapr.io/config: "pipeline-config"
```

## Mounting the WASM File

When running in Kubernetes, mount the WASM file via ConfigMap or an init container:

```yaml
initContainers:
  - name: wasm-loader
    image: myregistry/wasm-middleware:latest
    command: ["cp", "/middleware.wasm", "/components/middleware.wasm"]
    volumeMounts:
      - name: wasm-volume
        mountPath: /components
volumes:
  - name: wasm-volume
    emptyDir: {}
```

Mount the same volume into the `daprd` sidecar container using Dapr volume mounts if supported, or use a ConfigMap with binary data for small WASM files.

## Testing the WASM Middleware

Run locally with Dapr self-hosted mode:

```bash
dapr run \
  --app-id wasm-test \
  --resources-path ./components \
  --config ./config.yaml \
  -- go run ./testserver/main.go

# Send a test request
curl -X POST http://localhost:3500/v1.0/invoke/wasm-test/method/process \
  -H "Content-Type: application/json" \
  -d '{"data": "test"}'
```

## Summary

Dapr supports WASM middleware via the `middleware.http.wasm` component type, enabling portable, sandboxed custom logic inside the sidecar. Write WASM modules using TinyGo or AssemblyScript, mount them via init containers or ConfigMaps in Kubernetes, and chain them into the Dapr HTTP pipeline via Configuration. This approach extends Dapr's capabilities without adding new services.
