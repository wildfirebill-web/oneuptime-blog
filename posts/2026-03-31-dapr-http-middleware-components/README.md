# How to Develop Dapr HTTP Middleware Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, HTTP Middleware, Pluggable Component, Security, Extension

Description: Build custom Dapr HTTP middleware components to add authentication, request transformation, or rate limiting to the Dapr sidecar pipeline.

---

## Dapr HTTP Middleware Pipeline

Dapr supports middleware components that intercept HTTP requests flowing through the sidecar. Built-in middleware includes OAuth2, rate limiting, and request routing. Custom HTTP middleware lets you add organization-specific logic - custom auth schemes, header transformation, request logging - to the Dapr pipeline without changing application code.

## Middleware Architecture

Dapr's HTTP middleware pipeline processes requests in order:

```
App -> Dapr HTTP Port (3500) -> [Middleware 1] -> [Middleware 2] -> App Handler
                                                                         |
                                                              Service Invocation / APIs
```

## Implementing a Custom Middleware Component

Dapr HTTP middleware uses Go's `fasthttp` middleware pattern. Create a middleware that adds custom authentication headers:

```go
package main

import (
    "github.com/dapr/dapr/pkg/middleware"
    "github.com/valyala/fasthttp"
)

// CustomAuthMiddleware validates a custom token header
func NewCustomAuthMiddleware(metadata middleware.Metadata) (middleware.FastHTTPMiddleware, error) {
    secret := metadata.Properties["secret"]

    return func(next fasthttp.RequestHandler) fasthttp.RequestHandler {
        return func(ctx *fasthttp.RequestCtx) {
            token := string(ctx.Request.Header.Peek("X-Custom-Token"))

            if token != secret {
                ctx.Response.SetStatusCode(fasthttp.StatusUnauthorized)
                ctx.Response.SetBodyString(`{"error": "unauthorized"}`)
                return
            }

            // Add a downstream header with verified identity
            ctx.Request.Header.Set("X-Verified-User", "service-account")
            next(ctx)
        }
    }, nil
}
```

## Registering the Middleware Component

```go
package main

import (
    "github.com/dapr/dapr/pkg/components"
    httpMiddleware "github.com/dapr/dapr/pkg/middleware/http"
)

func init() {
    components.RegisterHTTPMiddleware("middleware.http.custom-auth",
        func(metadata middleware.Metadata) (middleware.FastHTTPMiddleware, error) {
            return NewCustomAuthMiddleware(metadata)
        },
    )
}
```

## Building a Request Logger Middleware

```go
func NewRequestLoggerMiddleware(metadata middleware.Metadata) (middleware.FastHTTPMiddleware, error) {
    logLevel := metadata.Properties["logLevel"]

    return func(next fasthttp.RequestHandler) fasthttp.RequestHandler {
        return func(ctx *fasthttp.RequestCtx) {
            start := time.Now()

            // Process request
            next(ctx)

            duration := time.Since(start)
            if logLevel == "info" {
                log.Printf("[MIDDLEWARE] %s %s -> %d (%s)",
                    ctx.Method(),
                    ctx.Path(),
                    ctx.Response.StatusCode(),
                    duration,
                )
            }
        }
    }, nil
}
```

## Component and Configuration YAML

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: custom-auth
spec:
  type: middleware.http.custom-auth
  version: v1
  metadata:
    - name: secret
      secretKeyRef:
        name: middleware-secret
        key: token
```

Apply middleware in the Dapr Configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  httpPipeline:
    handlers:
      - name: custom-auth
        type: middleware.http.custom-auth
      - name: uppercase
        type: middleware.http.uppercase
```

## Rate Limiting Middleware

```go
func NewRateLimitMiddleware(metadata middleware.Metadata) (middleware.FastHTTPMiddleware, error) {
    maxRPS, _ := strconv.Atoi(metadata.Properties["maxRequestsPerSecond"])
    limiter := rate.NewLimiter(rate.Limit(maxRPS), maxRPS)

    return func(next fasthttp.RequestHandler) fasthttp.RequestHandler {
        return func(ctx *fasthttp.RequestCtx) {
            if !limiter.Allow() {
                ctx.Response.SetStatusCode(fasthttp.StatusTooManyRequests)
                ctx.Response.SetBodyString(`{"error": "rate limit exceeded"}`)
                return
            }
            next(ctx)
        }
    }, nil
}
```

## Testing Middleware Locally

```bash
dapr run \
  --app-id my-app \
  --app-port 8080 \
  --config ./config/appconfig.yaml \
  --components-path ./components \
  -- go run main.go

# Test middleware is applied
curl -H "X-Custom-Token: wrong-token" http://localhost:3500/v1.0/invoke/target/method/test
# Expected: 401 Unauthorized

curl -H "X-Custom-Token: correct-token" http://localhost:3500/v1.0/invoke/target/method/test
# Expected: 200 OK
```

## Summary

Dapr HTTP middleware components provide a powerful extension point for cross-cutting concerns like authentication, rate limiting, and request transformation. By implementing the fasthttp middleware pattern and registering it as a Dapr component, you can add organizational security policies or observability logic to the Dapr pipeline declaratively through Configuration manifests.
