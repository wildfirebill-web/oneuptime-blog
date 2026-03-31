# How to Audit Secret Access Patterns with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Security, Audit, Secret Stores, Observability

Description: Learn how to audit and monitor secret access patterns in Dapr applications using middleware, distributed tracing, and structured logging to detect unauthorized or anomalous usage.

---

## Why Audit Secret Access

In regulated environments and security-conscious organizations, knowing which services accessed which secrets, when, and how often is essential for compliance and threat detection. Dapr's sidecar architecture provides natural instrumentation points for capturing this data without modifying application code.

## Enabling Distributed Tracing for Secret Access

Dapr automatically generates spans for secret store operations when tracing is enabled. Configure the Zipkin or OTLP exporter:

```yaml
# components/tracing.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprconfig
  namespace: default
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://zipkin.monitoring.svc.cluster.local:9411/api/v2/spans"
```

Apply this configuration to your app:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "my-service"
        dapr.io/config: "daprconfig"
```

With tracing enabled, every call to `GET /v1.0/secrets/{storeName}/{key}` generates a trace span with metadata including the store name, key, and app ID.

## Custom Audit Middleware

Build a Dapr HTTP middleware component that logs all secret access:

```go
// middleware/secret_audit.go
package middleware

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    "time"

    "go.opentelemetry.io/otel/trace"
)

type SecretAccessEvent struct {
    Timestamp  string `json:"timestamp"`
    AppID      string `json:"app_id"`
    SecretStore string `json:"secret_store"`
    SecretKey  string `json:"secret_key"`
    TraceID    string `json:"trace_id"`
    StatusCode int    `json:"status_code"`
    DurationMs int64  `json:"duration_ms"`
}

func SecretAuditMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Only intercept secret API calls
        if !strings.HasPrefix(r.URL.Path, "/v1.0/secrets/") {
            next.ServeHTTP(w, r)
            return
        }

        start := time.Now()
        appID := r.Header.Get("dapr-app-id")

        // Parse path: /v1.0/secrets/{storeName}/{key}
        parts := strings.Split(r.URL.Path, "/")
        storeName, secretKey := "", ""
        if len(parts) >= 5 {
            storeName = parts[4]
        }
        if len(parts) >= 6 {
            secretKey = parts[5]
        }

        // Wrap response writer to capture status code
        rw := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(rw, r)

        // Extract trace ID from context
        traceID := ""
        if span := trace.SpanFromContext(r.Context()); span != nil {
            traceID = span.SpanContext().TraceID().String()
        }

        event := SecretAccessEvent{
            Timestamp:   start.UTC().Format(time.RFC3339),
            AppID:       appID,
            SecretStore: storeName,
            SecretKey:   secretKey,
            TraceID:     traceID,
            StatusCode:  rw.statusCode,
            DurationMs:  time.Since(start).Milliseconds(),
        }

        data, _ := json.Marshal(event)
        fmt.Printf("[SECRET_AUDIT] %s\n", data)

        // In production, send to your SIEM or log aggregator
        sendToAuditLog(context.Background(), event)
    })
}
```

## Structured Audit Logging in Application Code

If you prefer application-level auditing, wrap your Dapr client calls:

```python
import json
import logging
import time
from dapr.clients import DaprClient
from datetime import datetime

logger = logging.getLogger("secret_audit")

class AuditedDaprClient:
    def __init__(self, app_id: str):
        self.app_id = app_id
        self.client = DaprClient()

    def get_secret(self, store_name: str, key: str) -> dict:
        start = time.time()
        success = False
        try:
            result = self.client.get_secret(store_name=store_name, key=key)
            success = True
            return result
        except Exception as e:
            logger.error("Secret access failed", extra={
                "store": store_name,
                "key": key,
                "error": str(e)
            })
            raise
        finally:
            duration_ms = (time.time() - start) * 1000
            logger.info("secret_access", extra={
                "event_type": "secret_access",
                "app_id":     self.app_id,
                "store_name": store_name,
                "secret_key": key,
                "success":    success,
                "duration_ms": round(duration_ms, 2),
                "timestamp":  datetime.utcnow().isoformat(),
            })

# Usage
client = AuditedDaprClient(app_id="payment-service")
creds = client.get_secret("vault", "database/credentials")
```

## Analyzing Secret Access Patterns

Query your audit logs to detect anomalous patterns:

```bash
# Find which apps accessed which secrets in the last hour (using jq on JSON logs)
cat audit.log | jq -r 'select(.event_type == "secret_access") |
  "\(.app_id) accessed \(.store_name)/\(.secret_key) at \(.timestamp)"' | \
  sort | uniq -c | sort -rn | head -20
```

```bash
# Detect unusual access frequency - more than 100 accesses in 5 minutes
cat audit.log | jq -r '
  select(.event_type == "secret_access") |
  "\(.app_id):\(.store_name):\(.secret_key):\(.timestamp[0:16])"
' | sort | uniq -c | awk '$1 > 100' | sort -rn
```

## Prometheus Metrics for Secret Access

Export metrics to track secret access rates:

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var secretAccessCounter = promauto.NewCounterVec(
    prometheus.CounterOpts{
        Name: "dapr_secret_access_total",
        Help: "Total number of Dapr secret accesses",
    },
    []string{"app_id", "store", "key", "status"},
)

var secretAccessDuration = promauto.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "dapr_secret_access_duration_seconds",
        Help:    "Duration of Dapr secret access operations",
        Buckets: prometheus.DefBuckets,
    },
    []string{"app_id", "store"},
)

// Record metrics in your middleware or wrapper
secretAccessCounter.WithLabelValues(appID, storeName, secretKey, status).Inc()
```

## Summary

Auditing secret access in Dapr can be implemented at multiple levels: Dapr's distributed tracing captures all secret API calls automatically, custom HTTP middleware can enrich logs with application context, and application-level wrappers give fine-grained control over what gets logged. Combining structured audit logs with Prometheus metrics and trace correlation gives security and compliance teams full visibility into which services are accessing which secrets, enabling both real-time alerting and retroactive forensics.
