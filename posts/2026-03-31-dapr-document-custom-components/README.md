# How to Document Custom Dapr Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Documentation, Custom Component, README, Metadata

Description: Learn how to document custom Dapr components with metadata specs, configuration examples, and operational runbooks that match the official Dapr component documentation standard.

---

## Why Good Documentation Matters

A Dapr component without clear documentation creates friction for every team that uses it. Good documentation covers metadata configuration, authentication requirements, operational limits, version compatibility, and troubleshooting steps. Following the official Dapr component documentation structure makes your component feel native to the ecosystem.

## Component README Structure

Follow the format used by official Dapr components:

```markdown
# My Custom State Store

## Component format

To set up the custom state store, create a Component resource of type `state.mystore`:

    apiVersion: dapr.io/v1alpha1
    kind: Component
    metadata:
      name: statestore
    spec:
      type: state.mystore
      version: v2
      metadata:
      - name: connectionString
        value: "host=mydb port=5432 user=dapr password=secret dbname=appstate"
      - name: compressionEnabled
        value: "true"
      - name: maxConnections
        value: "10"
      - name: ttlInSeconds
        value: "3600"

## Spec metadata fields

| Field | Required | Details | Example |
|---|---|---|---|
| connectionString | Y | PostgreSQL connection string | host=localhost port=5432 ... |
| compressionEnabled | N | Enable LZ4 compression for stored values. Default: false | "true" |
| maxConnections | N | Connection pool size. Default: 5 | "10" |
| ttlInSeconds | N | Default TTL for keys in seconds. 0 means no expiry. Default: 0 | "3600" |

## Authentication

The component supports the following authentication methods:
- Connection string with username/password
- IAM role-based auth (AWS RDS IAM)
- mTLS with client certificates

## Related links
- [Dapr state management overview](https://docs.dapr.io/developing-applications/building-blocks/state-management/)
- [How-To: Save and get state](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-get-save-state/)
```

## Generating Metadata Documentation from Code

Use Go struct tags to auto-generate metadata docs:

```go
type MyStoreMetadata struct {
    ConnectionString  string `mapstructure:"connectionString" mdonly:"false" mdesc:"PostgreSQL connection string"`
    CompressionEnabled bool  `mapstructure:"compressionEnabled" mdonly:"false" mdesc:"Enable LZ4 value compression. Default: false"`
    MaxConnections    int    `mapstructure:"maxConnections" mdonly:"false" mdesc:"Connection pool size. Default: 5"`
    TTLInSeconds      int    `mapstructure:"ttlInSeconds" mdonly:"false" mdesc:"Default TTL seconds. 0 = no expiry"`
}
```

Generate the metadata table:

```bash
# Using the Dapr metadata documentation generator
go run github.com/dapr/components-contrib/tools/generate-docs \
  --component state.mystore \
  --output ./docs/state/mystore.md
```

## API Endpoint Documentation

Document any admin or diagnostic endpoints your component exposes:

```markdown
## Admin endpoints

### GET /health

Returns component health status.

**Response**:
    {
      "status": "healthy",
      "version": "v2.1.0",
      "connections": {"active": 3, "idle": 2, "max": 10}
    }

### POST /admin/flush

Clears all stored state. Requires `X-Admin-Token` header.

**Warning**: This operation is irreversible.
```

## Troubleshooting Section

Always include a troubleshooting table:

```markdown
## Troubleshooting

| Error | Cause | Resolution |
|---|---|---|
| `connection refused` | DB not reachable | Check `connectionString` host and port |
| `authentication failed` | Wrong credentials | Verify username/password in secret |
| `socket not found` | Component not started | Ensure component pod is running |
| `version mismatch` | Wrong YAML version field | Set `version: v2` to match component |
```

## Summary

Documenting custom Dapr components to the same standard as official components reduces onboarding time and support requests. A well-structured README with a metadata spec table, authentication options, and a troubleshooting guide makes the component self-service. Auto-generating metadata documentation from Go struct tags keeps docs synchronized with implementation and reduces the chance of documentation drift.
