# How to Audit Secret Access in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Audit, Secret Management, Security, Observability

Description: Learn how to track and audit which Dapr services are accessing which secrets using logging, tracing, and backend-level audit features.

---

Knowing which services accessed which secrets and when is essential for security compliance. Dapr provides several layers where you can capture this information: the Dapr sidecar logs, distributed tracing, and the audit features of the underlying secret backend.

## Layer 1: Dapr Sidecar Logs

The Dapr sidecar logs every secret API call. Set the log level to `info` or `debug` to capture these events:

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
        dapr.io/log-level: "info"
```

Secret access appears in the sidecar logs:

```bash
kubectl logs deployment/my-service -c daprd | grep "secret"
```

Sample log output:

```
time="2026-03-31T10:23:45Z" level=info msg="SECRET: GET secret name=db-creds, store=k8s-store" app_id=my-service
```

## Layer 2: Distributed Tracing for Secret Calls

Enable OpenTelemetry tracing in your Dapr configuration to trace secret API calls:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://jaeger-collector:9411/api/v2/spans"
```

Each secret retrieval generates a span with the app ID, secret store name, and secret key, giving you a full trace of which service fetched which secret and when.

## Layer 3: HashiCorp Vault Audit Logging

When using Vault as the backend, enable a file audit device:

```bash
vault audit enable file file_path=/vault/logs/audit.log
```

Every secret read is logged:

```json
{
  "time": "2026-03-31T10:23:45.123456789Z",
  "type": "response",
  "auth": {
    "display_name": "dapr-my-service",
    "policies": ["db-read-policy"]
  },
  "request": {
    "operation": "read",
    "path": "secret/data/db-creds"
  }
}
```

## Layer 4: Kubernetes Audit Logs for Secret Reads

For Kubernetes secret stores, enable Kubernetes API server audit logging and filter for secret reads:

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["get", "list"]
```

Apply the policy to your API server configuration and ship logs to your SIEM.

## Correlating Audit Events

Use the Dapr app ID to correlate sidecar logs with backend audit logs:

```bash
# Find all secret accesses by a specific service in the last hour
kubectl logs -l app=my-service -c daprd --since=1h | grep "SECRET:"
```

Then cross-reference with Vault audit logs filtering on the same time window and the Vault role bound to that service account.

## Summary

Dapr secret access can be audited at multiple layers: sidecar logs capture every API call, distributed tracing records them as spans, and backend stores like Vault and Kubernetes provide their own audit trails. Combining these sources gives you complete visibility into which service accessed which secret and when, satisfying compliance requirements for secret access auditing.
