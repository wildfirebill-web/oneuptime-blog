# How to Configure Dapr Environment Variables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Environment Variable, Configuration, Sidecar, Kubernetes

Description: Learn how to configure Dapr sidecar behavior using environment variables for app IDs, ports, logging levels, and component paths.

---

## Overview

Dapr's sidecar (daprd) behavior can be controlled through environment variables and CLI flags. Understanding which variables affect the sidecar helps you tune logging, component discovery, and API behavior across different deployment environments.

## Core Sidecar Environment Variables

The following environment variables are automatically set by the Dapr operator when injecting the sidecar:

| Variable | Purpose | Example |
|---|---|---|
| `DAPR_HTTP_PORT` | Sidecar HTTP port | `3500` |
| `DAPR_GRPC_PORT` | Sidecar gRPC port | `50001` |
| `APP_ID` | Application identifier | `my-service` |
| `NAMESPACE` | Kubernetes namespace | `default` |

Access them in your application:

```python
import os

dapr_http_port = os.getenv("DAPR_HTTP_PORT", "3500")
app_id = os.getenv("APP_ID", "local-app")
dapr_base_url = f"http://localhost:{dapr_http_port}/v1.0"
print(f"Connecting to Dapr at {dapr_base_url} as {app_id}")
```

## Setting Custom Environment Variables via Annotations

Inject environment variables into the Dapr sidecar container:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "myservice"
  dapr.io/sidecar-env: "LOG_LEVEL=debug,OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317"
```

## Configuring Logging Level

Control Dapr log verbosity:

```yaml
annotations:
  dapr.io/log-level: "debug"
  dapr.io/log-as-json: "true"
```

Or via environment variable in the Helm chart:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set global.logLevel=info
```

## App-Level Environment Variables

Pass secrets and config values to your application container alongside Dapr:

```yaml
spec:
  containers:
  - name: myapp
    image: myrepo/myapp:latest
    env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: host
    - name: DAPR_HTTP_PORT
      value: "3500"
```

## Using ConfigMaps for Environment Configuration

Manage non-sensitive configuration centrally:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DAPR_HTTP_PORT: "3500"
  APP_ENV: "production"
  LOG_LEVEL: "info"
```

Reference in your Deployment:

```yaml
envFrom:
- configMapRef:
    name: app-config
```

## Verifying Environment Variables at Runtime

Inspect the sidecar environment in a running pod:

```bash
kubectl exec -it <pod-name> -c daprd -- env | grep -E "DAPR|APP_|NAMESPACE"
```

## Summary

Dapr automatically injects key environment variables like `DAPR_HTTP_PORT` and `APP_ID` into application containers. Use the `dapr.io/sidecar-env` annotation to pass custom variables to the sidecar, and use standard Kubernetes ConfigMaps and Secrets for application-level configuration to maintain clean separation between app config and Dapr tuning.
