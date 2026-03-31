# How to Use Middleware for Request Transformation in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Middleware, Request Transformation, HTTP, Pipeline

Description: Learn how to use Dapr middleware components to transform HTTP requests and responses, including header manipulation, body transformation, and uppercase middleware.

---

## Introduction

Dapr middleware can transform requests before they reach your application and transform responses before they are returned to the caller. The built-in uppercase middleware is a simple example, but Wasm and custom middleware enable sophisticated transformations like JSON schema validation, payload enrichment, and protocol adaptation.

## Built-in Uppercase Middleware

The uppercase middleware transforms request bodies to uppercase:

```yaml
# components/uppercase.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: uppercase
spec:
  type: middleware.http.uppercase
  version: v1
```

```yaml
# config/uppercase-pipeline.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: uppercase-pipeline
spec:
  httpPipeline:
    handlers:
      - name: uppercase
        type: middleware.http.uppercase
```

## Request Header Transformation with Wasm

Write a Wasm middleware in Go/TinyGo to inject headers:

```go
// wasm/inject_headers.go
package main

import (
    "strings"
)

//export handle_request
func handle_request() int32 {
    // Inject correlation ID header
    setRequestHeader("X-Correlation-ID", generateCorrelationID())
    setRequestHeader("X-Processed-By", "dapr-middleware")
    return 0 // Continue processing
}

func main() {}
```

## Component for Wasm Transformation

```yaml
# components/transform-wasm.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: transform-wasm
spec:
  type: middleware.http.wasm
  version: v1
  metadata:
    - name: url
      value: "file://./wasm/inject_headers.wasm"
```

## Router Alias for Path Normalization

Use router alias to normalize API paths:

```yaml
# components/path-normalizer.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: path-normalizer
spec:
  type: middleware.http.routeralias
  version: v1
  metadata:
    - name: routes
      value: |
        {
          "/v1/order":  "/api/orders",
          "/v1/orders": "/api/orders",
          "/order":     "/api/orders"
        }
```

## Combining Transformations in a Pipeline

```yaml
# config/transform-pipeline.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: transform-pipeline
spec:
  httpPipeline:
    handlers:
      - name: path-normalizer
        type: middleware.http.routeralias
      - name: transform-wasm
        type: middleware.http.wasm
      - name: ratelimit
        type: middleware.http.ratelimit
```

## Testing Request Transformation

```python
# app.py - your application receives the transformed request
from flask import Flask, request

app = Flask(__name__)

@app.route("/api/orders", methods=["POST"])
def create_order():
    # These headers were injected by Dapr middleware
    correlation_id = request.headers.get("X-Correlation-ID", "none")
    processed_by = request.headers.get("X-Processed-By", "none")
    print(f"Correlation ID: {correlation_id}")
    print(f"Processed By: {processed_by}")
    return {"status": "ok", "correlation_id": correlation_id}
```

```bash
dapr run \
  --app-id transform-service \
  --app-port 5000 \
  --config ./config/transform-pipeline.yaml \
  --components-path ./components \
  -- flask run --port 5000

# Call the legacy path - gets normalized to /api/orders
curl -X POST \
  http://localhost:3500/v1.0/invoke/transform-service/method/v1/order \
  -H "Content-Type: application/json" \
  -d '{"item":"widget"}'
```

## Response Transformation

For response transformation, use a Wasm module that intercepts the response:

```go
//export handle_response
func handle_response(status_code int32) int32 {
    if status_code == 200 {
        setResponseHeader("X-Cache-Control", "max-age=60")
    }
    return status_code
}
```

## Summary

Dapr middleware provides multiple options for request and response transformation: the built-in uppercase and router alias components handle simple cases, while Wasm middleware enables custom logic in any Wasm-compatible language. Chaining transformations in the pipeline lets you normalize paths, inject headers, validate payloads, and modify responses without touching application code.
