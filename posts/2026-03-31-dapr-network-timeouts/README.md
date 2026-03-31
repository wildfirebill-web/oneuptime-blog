# How to Configure Dapr Network Timeouts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Network, Timeout, Resiliency, Configuration

Description: Learn how to configure network-level timeouts in Dapr for service invocation, pub/sub delivery, and binding operations to prevent resource exhaustion.

---

## Network Timeouts in Dapr

Network timeouts prevent slow downstream services or unresponsive brokers from consuming connection pools and goroutines indefinitely. Dapr exposes timeout configuration at multiple levels: per-target resiliency policies, global sidecar settings, and per-request override headers. Understanding which timeout applies where prevents subtle bugs in production.

## Resiliency-Based Timeouts (Per Target)

The most common approach is setting timeouts in a Resiliency policy:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: app-resiliency
  namespace: production
spec:
  policies:
    timeouts:
      fast: 2s
      standard: 10s
      slow: 60s

  targets:
    apps:
      payment-service:
        timeout: fast
      report-generator:
        timeout: slow
      inventory-service:
        timeout: standard
    components:
      orders-pubsub:
        outbound:
          timeout: standard
      file-storage:
        outbound:
          timeout: slow
```

## Sidecar-Level Network Timeouts

Configure global HTTP client timeouts for the Dapr sidecar via Helm values or annotations:

```yaml
# Helm values for Dapr system chart
dapr_sidecar_injector:
  sidecarContainerImage: daprio/daprd:1.14.0
  injectedSideCarContainerSettings:
    env:
    - name: DAPR_HTTP_CLIENT_TIMEOUT
      value: "30"
    - name: DAPR_GRPC_KEEPALIVE_TIMEOUT
      value: "10"
```

Or via pod annotations:

```yaml
annotations:
  dapr.io/config: "app-config"
  dapr.io/http-read-buffer-size: "4096"
  dapr.io/http-max-request-size: "10"
```

## Per-Request Timeout Override

Pass a timeout for individual calls using the `dapr-timeout-ms` metadata header:

```python
from dapr.clients import DaprClient
import json

def call_with_custom_timeout(app_id: str, method: str, data: dict, timeout_ms: int):
    with DaprClient() as client:
        response = client.invoke_method(
            app_id=app_id,
            method_name=method,
            data=json.dumps(data),
            content_type="application/json",
            metadata=(("dapr-timeout-ms", str(timeout_ms)),)
        )
        return json.loads(response.data)

# Use a 3-second timeout for this specific call
result = call_with_custom_timeout("slow-service", "process", payload, 3000)
```

## Timeout Hierarchy

Dapr applies timeouts in the following order (most specific wins):

1. Per-request `dapr-timeout-ms` metadata
2. Resiliency policy timeout for the specific target
3. Global sidecar HTTP client timeout

## Monitoring Timeout Errors

Use Dapr Prometheus metrics to track timeout rates:

```bash
# Count timeout responses (408, 504)
curl -s http://localhost:9090/api/v1/query \
  --data-urlencode 'query=sum(rate(dapr_http_client_completed_count{status_code=~"408|504"}[5m])) by (app_id)'
```

Alert when timeout rate spikes:

```yaml
groups:
- name: dapr-timeouts
  rules:
  - alert: HighTimeoutRate
    expr: rate(dapr_http_client_completed_count{status_code="408"}[5m]) > 0.1
    for: 3m
    labels:
      severity: warning
    annotations:
      summary: "High timeout rate on {{ $labels.app_id }}"
```

## Summary

Dapr network timeouts should be configured at the resiliency policy level for per-target control, with global sidecar settings as a safety net. Per-request timeout overrides handle special cases like user-initiated long-running operations. Prometheus metrics on 408 and 504 status codes provide real-time visibility into timeout rates and trigger alerts before resource exhaustion occurs.
