# How to Use Wasm Middleware in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Middleware, WebAssembly, Wasm, HTTP

Description: Learn how to configure the Dapr Wasm middleware to run custom request and response transformation logic compiled to WebAssembly in the Dapr HTTP pipeline.

---

## Introduction

The Dapr Wasm middleware (`middleware.http.wasm`) executes WebAssembly modules in the HTTP request pipeline. This lets you write custom middleware logic in any language that compiles to Wasm (Go, Rust, C, TinyGo) and run it inside the Dapr sidecar without modifying your application.

## Use Cases

- Request header injection or validation
- Response body transformation
- Custom authentication logic
- Request logging and tracing enrichment

## Writing a Wasm Middleware in TinyGo

```go
// middleware/main.go
package main

import (
    "fmt"
    "unsafe"
)

//export malloc
func malloc(size uint32) uint32 {
    buf := make([]byte, size)
    return uint32(uintptr(unsafe.Pointer(&buf[0])))
}

//export handle_request
func handle_request(ptr, size uint32) uint64 {
    // Read incoming request bytes
    request := readMemory(ptr, size)
    fmt.Printf("Handling request: %d bytes\n", len(request))

    // Add a custom header by modifying the request
    modified := addHeader(request, "X-Processed-By", "dapr-wasm")

    return writeResponse(modified)
}

func main() {}
```

## Compiling to Wasm with TinyGo

```bash
tinygo build \
  -o middleware.wasm \
  -target=wasi \
  ./middleware/
```

## Using Rust for Wasm Middleware

```rust
// src/lib.rs
use std::collections::HashMap;

#[no_mangle]
pub fn handle_request(ptr: *mut u8, len: usize) -> u64 {
    // Read request bytes
    let request = unsafe {
        std::slice::from_raw_parts(ptr, len)
    };

    // Add custom header
    println!("Processing request: {} bytes", request.len());
    0 // Return 0 to continue processing
}
```

```bash
cargo build --target wasm32-wasi --release
```

## Component Configuration

```yaml
# components/wasm-middleware.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: wasm-middleware
spec:
  type: middleware.http.wasm
  version: v1
  metadata:
    - name: url
      value: "file://./middleware/middleware.wasm"
```

## Loading Wasm from HTTP

```yaml
metadata:
  - name: url
    value: "https://my-storage.example.com/middleware.wasm"
```

## Pipeline Configuration

```yaml
# config/wasm-pipeline.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: wasm-pipeline
spec:
  httpPipeline:
    handlers:
      - name: wasm-middleware
        type: middleware.http.wasm
```

## Running the App

```bash
dapr run \
  --app-id wasm-service \
  --app-port 8080 \
  --config ./config/wasm-pipeline.yaml \
  --components-path ./components \
  -- python app.py
```

## Testing the Wasm Middleware

```bash
curl -v http://localhost:3500/v1.0/invoke/wasm-service/method/hello

# Check that the custom header was injected
# < X-Processed-By: dapr-wasm
```

## Benefits of Wasm Middleware

- Language-agnostic: write in Go, Rust, C, or AssemblyScript
- Sandboxed execution: Wasm runs in a secure sandbox
- High performance: near-native execution speed
- Portable: the same `.wasm` file runs on any platform

## Summary

Dapr Wasm middleware extends the sidecar pipeline with custom WebAssembly logic. Compile your middleware in any Wasm-compatible language, reference the binary in the component YAML, and attach it to the HTTP pipeline. This provides a language-agnostic, sandboxed way to add custom request processing without modifying your application or the Dapr sidecar source code.
