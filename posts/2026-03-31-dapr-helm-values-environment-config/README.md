# How to Use Helm Values for Dapr Environment Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Helm, Configuration, Kubernetes, DevOps

Description: Learn how to use Helm values files to manage environment-specific Dapr component configuration for dev, staging, and production deployments.

---

Helm is widely used for packaging and deploying Kubernetes applications. When you deploy Dapr components alongside your application, Helm values files give you a clean way to parameterize component configuration per environment without duplicating YAML templates.

## Chart Structure

```
mychart/
  Chart.yaml
  values.yaml
  values-dev.yaml
  values-production.yaml
  templates/
    statestore.yaml
    pubsub.yaml
    deployment.yaml
```

## Define a Parameterized Component Template

Use Helm template syntax to reference values from the values file.

```yaml
# templates/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: {{ .Values.namespace }}
spec:
  type: {{ .Values.statestore.type }}
  version: v1
  metadata:
    - name: redisHost
      value: "{{ .Values.statestore.redisHost }}"
    {{- if .Values.statestore.passwordSecret }}
    - name: redisPassword
      secretKeyRef:
        name: {{ .Values.statestore.passwordSecret }}
        key: password
    {{- end }}
```

## Set Default Values

The base `values.yaml` provides defaults suitable for local or dev use.

```yaml
# values.yaml
namespace: default
statestore:
  type: state.in-memory
  redisHost: ""
  passwordSecret: ""
pubsub:
  type: pubsub.in-memory
  brokerUrl: ""
```

## Override for Each Environment

Create a values file for each environment that overrides only what differs.

```yaml
# values-production.yaml
namespace: production
statestore:
  type: state.redis
  redisHost: "redis-master.production.svc.cluster.local:6379"
  passwordSecret: "redis-secret"
pubsub:
  type: pubsub.kafka
  brokerUrl: "kafka-broker.production.svc.cluster.local:9092"
```

## Deploy with Environment-Specific Values

Pass the environment values file when installing or upgrading:

```bash
# Deploy to dev (uses defaults)
helm upgrade --install myapp ./mychart \
  --namespace dev \
  --create-namespace

# Deploy to production
helm upgrade --install myapp ./mychart \
  --namespace production \
  --create-namespace \
  -f values-production.yaml
```

## Inject Secrets Separately

Avoid putting secrets directly in values files. Use Helm's `--set` flag or a secrets manager like External Secrets Operator to inject sensitive values at deploy time.

```bash
helm upgrade --install myapp ./mychart \
  -f values-production.yaml \
  --set statestore.redisHost="$(kubectl get secret redis-config -o jsonpath='{.data.host}' | base64 -d)"
```

## Template the Dapr Configuration CRD

Include the Dapr Configuration CRD in your chart and parameterize tracing and resiliency settings.

```yaml
# templates/dapr-config.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
  namespace: {{ .Values.namespace }}
spec:
  tracing:
    samplingRate: "{{ .Values.dapr.tracingSampleRate }}"
```

```yaml
# values-production.yaml (additional key)
dapr:
  tracingSampleRate: "1"
```

## Summary

Helm values files provide a structured and repeatable way to configure Dapr components across environments. By combining parameterized templates with environment-specific values files, you keep your chart DRY while giving each environment the exact configuration it needs.
