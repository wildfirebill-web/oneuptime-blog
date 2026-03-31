# How to Optimize Dapr HTTP Communication Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, HTTP, Performance, Microservice, Networking

Description: Improve Dapr HTTP service invocation performance through connection reuse, request pipelining, middleware tuning, and proper timeout configuration.

---

## Dapr HTTP Communication Fundamentals

When Dapr services communicate via HTTP, requests flow from your app through the local sidecar proxy to the target sidecar, then to the destination app. Each hop adds latency, so minimizing overhead at every layer is critical for performance.

## Reuse HTTP Connections with Keep-Alive

Avoid the TCP handshake overhead on every request by using persistent connections. In Node.js:

```javascript
const http = require('http');
const { DaprClient } = require('@dapr/dapr');

const agent = new http.Agent({
    keepAlive: true,
    maxSockets: 50,
    keepAliveMsecs: 30000,
});

const client = new DaprClient({
    daprHost: '127.0.0.1',
    daprPort: '3500',
    httpEndpoint: 'http://127.0.0.1:3500',
    agent: agent,
});
```

In Python with httpx:

```python
import httpx

# Reusable client with connection pooling
async with httpx.AsyncClient(
    base_url="http://localhost:3500",
    limits=httpx.Limits(max_keepalive_connections=20, max_connections=50),
    timeout=5.0,
) as client:
    response = await client.post(
        "/v1.0/invoke/orderservice/method/getorder",
        json={"orderId": "123"},
    )
```

## Configure Dapr HTTP Timeout and Retry

Set appropriate timeouts to prevent cascading slowdowns. In Dapr resiliency policy:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: httpresiliency
spec:
  policies:
    timeouts:
      defaultTimeout: 3s
    retries:
      defaultRetry:
        policy: constant
        duration: 200ms
        maxRetries: 3
  targets:
    apps:
      orderservice:
        timeout: httpTimeout
        retry: defaultRetry
```

Apply it:

```bash
kubectl apply -f httpresiliency.yaml
```

## Enable HTTP/2 for the Dapr Sidecar

HTTP/2 multiplexing reduces head-of-line blocking. Force HTTP/2 in Dapr:

```yaml
annotations:
  dapr.io/app-protocol: "http"
  dapr.io/http2-enabled: "true"
  dapr.io/max-body-size: "32Mi"
  dapr.io/http-read-buffer-size: "32"
```

## Compress Large Response Payloads

Enable middleware-level compression for HTTP responses:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: httpmiddleware
spec:
  type: middleware.http.routeralias
  version: v1
  metadata:
    - name: routes
      value: |
        {
          "/api/large": "/v1.0/invoke/dataservice/method/large"
        }
```

For custom compression in Go:

```go
import "compress/gzip"

func compressedHandler(w http.ResponseWriter, r *http.Request) {
    if strings.Contains(r.Header.Get("Accept-Encoding"), "gzip") {
        w.Header().Set("Content-Encoding", "gzip")
        gz := gzip.NewWriter(w)
        defer gz.Close()
        w = &gzipResponseWriter{Writer: gz, ResponseWriter: w}
    }
    // handle request...
}
```

## Profile HTTP Performance with hey

Load-test Dapr HTTP endpoints to establish baselines:

```bash
# Install hey
go install github.com/rakyll/hey@latest

# Benchmark Dapr HTTP invocation
hey -n 10000 -c 100 \
  -m POST \
  -H "Content-Type: application/json" \
  -d '{"orderId":"123"}' \
  http://localhost:3500/v1.0/invoke/orderservice/method/getorder
```

## Summary

Optimizing Dapr HTTP communication requires persistent connection reuse, appropriate timeout and retry policies, HTTP/2 enablement, and payload compression. Using a load-testing tool like `hey` to benchmark before and after changes ensures your tuning efforts produce measurable improvements.
