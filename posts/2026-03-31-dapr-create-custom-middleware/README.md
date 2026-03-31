# How to Create Custom Middleware in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Middleware, Custom, Go, Extension

Description: Learn how to build and register a custom Dapr HTTP middleware component in Go, including request/response manipulation and pipeline integration.

---

## Introduction

Dapr middleware components are Go plugins that implement the `middleware.Middleware` interface. You can create custom middleware to add capabilities like custom authentication, request enrichment, response caching, or audit logging that are not available in the built-in middleware catalog.

## Middleware Interface

A Dapr HTTP middleware must implement:

```go
// The middleware handler function type
type MiddlewareFunc func(next http.Handler) http.Handler
```

## Project Structure

```text
custom-middleware/
  middleware/
    audit.go
  main.go
  go.mod
```

## Writing the Custom Middleware

```go
// middleware/audit.go
package middleware

import (
    "encoding/json"
    "log"
    "net/http"
    "time"
)

type AuditLog struct {
    Method     string    `json:"method"`
    Path       string    `json:"path"`
    StatusCode int       `json:"status_code"`
    Duration   string    `json:"duration_ms"`
    Timestamp  time.Time `json:"timestamp"`
}

type responseRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (r *responseRecorder) WriteHeader(code int) {
    r.statusCode = code
    r.ResponseWriter.WriteHeader(code)
}

// AuditMiddleware logs every request with its status code and duration
func AuditMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        recorder := &responseRecorder{
            ResponseWriter: w,
            statusCode:     http.StatusOK,
        }

        next.ServeHTTP(recorder, r)

        entry := AuditLog{
            Method:     r.Method,
            Path:       r.URL.Path,
            StatusCode: recorder.statusCode,
            Duration:   time.Since(start).String(),
            Timestamp:  start.UTC(),
        }
        data, _ := json.Marshal(entry)
        log.Printf("AUDIT: %s", string(data))
    })
}
```

## Registering the Middleware with Dapr Runtime

To use a custom middleware, you need to build a custom Dapr runtime. Create a component registration:

```go
// main.go
package main

import (
    "context"
    "github.com/dapr/dapr/cmd/daprd/main_windows"
    "github.com/dapr/kit/logger"
    mh "github.com/dapr/dapr/pkg/middleware/http"
    "my-module/middleware"
)

func main() {
    registry := mh.NewRegistry()
    registry.RegisterComponent(func(log logger.Logger) mh.Middleware {
        return mh.MiddlewareFunc(func(next http.Handler) http.Handler {
            return middleware.AuditMiddleware(next)
        })
    }, "audit-logger", "v1")

    // Start the Dapr runtime with the custom registry
    daprd.Start(context.Background(), registry)
}
```

## Simpler Approach - Sidecar HTTP Proxy Pattern

For most use cases, a simpler approach is to run your custom middleware as a proxy between Dapr and your app:

```go
// proxy/main.go
package main

import (
    "fmt"
    "log"
    "net/http"
    "net/http/httputil"
    "net/url"
    "os"
)

func main() {
    appURL, _ := url.Parse("http://localhost:8080")
    proxy := httputil.NewSingleHostReverseProxy(appURL)

    mux := http.NewServeMux()
    mux.Handle("/", auditWrapper(proxy))

    port := os.Getenv("PROXY_PORT")
    if port == "" {
        port = "8090"
    }
    log.Printf("Proxy listening on :%s", port)
    log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", port), mux))
}

func auditWrapper(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("[AUDIT] %s %s from %s", r.Method, r.URL.Path, r.RemoteAddr)
        next.ServeHTTP(w, r)
    })
}
```

## Using the Proxy with Dapr

```bash
# Point Dapr at the proxy, which forwards to your app
dapr run \
  --app-id my-service \
  --app-port 8090 \
  -- go run proxy/main.go &

go run app/main.go
```

## Custom Request Header Injection

```go
func HeaderInjectionMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        r.Header.Set("X-Request-ID", generateID())
        r.Header.Set("X-Service-Name", "my-service")
        next.ServeHTTP(w, r)
    })
}
```

## Summary

Custom Dapr middleware can be built either by extending the Dapr runtime with a registered component or by using a reverse proxy pattern that sits between Dapr and your application. The proxy pattern is simpler for most use cases and does not require building a custom Dapr binary. Both approaches give you full control over request and response manipulation at the sidecar layer.
