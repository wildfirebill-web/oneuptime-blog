# How to Understand Dapr Component Specification Format

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Component, Configuration, YAML, Specification

Description: Understand the Dapr component specification format including required fields, metadata structure, versioning, scopes, and how to validate component YAML files.

---

## What Is a Dapr Component Specification?

A Dapr component specification is a YAML file that tells the Dapr runtime how to connect to an external resource such as a state store, pub/sub broker, binding, or secret store. Components are the primary extension point in Dapr.

## Core Fields of a Component Spec

Every component YAML follows the same Kubernetes-style structure:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore          # Referenced in code and logs
  namespace: production     # Matches your app namespace
spec:
  type: state.redis         # Category.Implementation
  version: v1               # Component version
  initTimeout: 10s          # Max time to initialize
  ignoreErrors: false       # Fail sidecar if true
  metadata:
    - name: redisHost
      value: redis:6379
    - name: redisPassword
      secretKeyRef:
        name: redis-secret
        key: password
auth:
  secretStore: kubernetes
scopes:
  - service-a
  - service-b
```

## The `type` Field

The `type` field uses the format `category.implementation`:

| Category | Examples |
|----------|---------|
| state | state.redis, state.postgresql |
| pubsub | pubsub.kafka, pubsub.redis |
| bindings | bindings.kafka, bindings.http |
| secretstores | secretstores.kubernetes |
| crypto | crypto.azure.keyvault |

## Metadata Values and Secret References

Metadata values can be inline strings or references to secrets:

```yaml
metadata:
  - name: connectionString
    value: "host=postgres port=5432 dbname=mydb"

  # Or using a secret reference
  - name: connectionString
    secretKeyRef:
      name: db-secret
      key: connection-string
```

The `auth.secretStore` field tells Dapr which secret store to use for resolving `secretKeyRef` values.

## Using Scopes to Restrict Access

The `scopes` field limits which Dapr app IDs can use the component:

```yaml
scopes:
  - checkout-service
  - inventory-service
```

Without scopes, all apps in the namespace can access the component. This is important for security in multi-tenant clusters.

## Component Versioning

The `version` field in the spec refers to the component implementation version (not the Dapr runtime version). Most components are `v1`. Some newer components offer `v2` with breaking changes:

```yaml
spec:
  type: pubsub.kafka
  version: v1
```

Check the component's documentation page to see available versions and migration notes.

## Validating a Component File

Use the Dapr CLI to validate a component file before deploying:

```bash
dapr run --resources-path ./components -- echo "test"
```

For Kubernetes, apply with dry-run:

```bash
kubectl apply --dry-run=client -f components/statestore.yaml
```

Check Dapr sidecar logs for component initialization errors:

```bash
kubectl logs -l app=my-service -c daprd | grep "component\|error"
```

## Summary

A Dapr component specification uses a Kubernetes-style YAML structure with `apiVersion`, `kind`, `metadata`, `spec`, `auth`, and `scopes` fields. The `type` field uses a `category.implementation` format, metadata values can reference secrets, and scopes control which applications can access the component. Always validate component files with a dry-run before deploying to production.
