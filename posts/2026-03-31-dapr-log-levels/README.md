# How to Configure Dapr Log Levels

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Logging, Configuration, Debug, Kubernetes

Description: Control Dapr sidecar log verbosity by setting log levels per deployment to balance debugging visibility with log volume and storage costs.

---

Dapr supports five log levels that control how much output the sidecar produces. Choosing the right level for each environment - verbose for development, minimal for production - reduces log storage costs and makes it easier to find relevant entries.

## Available Log Levels

Dapr log levels from most to least verbose:

| Level | Use Case | Volume |
|-------|----------|--------|
| `debug` | Development troubleshooting | Very high |
| `info` | Normal production operation | Medium |
| `warn` | Production with warning visibility | Low |
| `error` | Minimal production logging | Very low |
| `fatal` | Emergency mode only | Minimal |

## Setting Log Level via Kubernetes Annotation

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "payment-service"
        dapr.io/log-level: "info"
```

Apply the change and restart the pod:

```bash
kubectl apply -f payment-service.yaml
kubectl rollout restart deployment/payment-service
```

## Setting Log Level for Self-Hosted Mode

```bash
# Run with debug logging
dapr run \
  --app-id myapp \
  --app-port 3000 \
  --log-level debug \
  -- python app.py

# Or use environment variable
DAPR_LOG_LEVEL=warn dapr run --app-id myapp --app-port 3000 -- python app.py
```

## Changing Log Level at Runtime

Dapr supports dynamic log level changes via the sidecar HTTP API without restarting:

```bash
# Change to debug level on a running sidecar
curl -X POST http://localhost:3500/v1.0/metadata \
  -H "Content-Type: application/json" \
  -d '{"level": "debug"}'

# Verify the current log level
curl http://localhost:3500/v1.0/metadata | jq '.extended.daprRuntimeVersion'
```

## Recommended Level by Environment

```yaml
# development/staging - use debug for full visibility
metadata:
  annotations:
    dapr.io/log-level: "debug"

# production - use info for standard operation
metadata:
  annotations:
    dapr.io/log-level: "info"

# high-volume production - use warn to reduce costs
metadata:
  annotations:
    dapr.io/log-level: "warn"
```

## What Each Level Logs

At `debug` level, Dapr logs every API call, state store operation, and pub/sub message:

```bash
kubectl logs deploy/order-service -c daprd | grep "DEBUG" | head -5
# time="2026-03-31T10:00:00Z" level=debug msg="HTTP API Called" method=GET url=/v1.0/state/statestore/order-123
```

At `info` level, you see component initialization and important lifecycle events:

```bash
kubectl logs deploy/order-service -c daprd | grep "INFO" | head -5
# time="2026-03-31T10:00:00Z" level=info msg="component loaded" component=statestore type=state.redis
```

At `warn` level, only non-fatal issues are logged:

```bash
kubectl logs deploy/order-service -c daprd | grep "WARN" | head -5
# time="2026-03-31T10:00:00Z" level=warning msg="retry attempt" attempt=2 component=pubsub
```

## Setting Global Default via Helm

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --reuse-values \
  --set global.logLevel=info \
  --set global.logAsJson=true
```

## Summary

Dapr log levels control sidecar verbosity from `debug` (all API calls) to `fatal` (only startup failures). Use `info` for standard production workloads and `warn` for high-volume services where log costs are a concern. For live troubleshooting, Dapr supports dynamic log level changes via the metadata API without pod restarts. Always pair log level configuration with JSON format (`dapr.io/log-as-json: "true"`) for structured log parsing.
