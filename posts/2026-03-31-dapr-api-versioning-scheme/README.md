# How to Understand Dapr API Versioning Scheme

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, API, Versioning, Compatibility, REST

Description: Understand how Dapr versions its HTTP and gRPC APIs, what version labels mean, and how to write code that stays compatible across Dapr upgrades.

---

## Dapr API Version Labels

Dapr uses URL-based versioning for its HTTP API. The current stable API is `v1.0`, with preview features under `v1.0-alpha1`. The pattern follows:

```
http://localhost:3500/v1.0/{feature}/{...}
http://localhost:3500/v1.0-alpha1/{preview-feature}/{...}
```

Examples:
- `GET /v1.0/state/{storeName}/{key}` - stable state API
- `POST /v1.0-alpha1/workflow/{workflowComponentName}/{workflowName}/start` - alpha workflow

## Stable vs Alpha vs Beta

| Label | Meaning |
|-------|---------|
| `v1.0` | Stable, backward-compatible within the major version |
| `v1.0-alpha1` | Feature preview - may change or be removed |
| `v1.0-rc1` | Release candidate - API is frozen but not yet stable |

Always check which stability tier an API belongs to before using it in production code.

## HTTP API Version Examples

Calling the state management API:

```bash
# Stable: Get state
curl http://localhost:3500/v1.0/state/statestore/mykey

# Stable: Save state
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "mykey", "value": {"name": "Alice"}}]'

# Alpha: Start a workflow
curl -X POST \
  http://localhost:3500/v1.0-alpha1/workflow/dapr/order-processing/start \
  -H "Content-Type: application/json" \
  -d '{"input": {"orderId": "123"}}'
```

## gRPC API Versioning

The Dapr gRPC API uses protobuf versioning in the package name:

```proto
package dapr.proto.runtime.v1;
```

The gRPC service is exposed on port 50001 by default. SDK clients abstract the version:

```go
client, err := dapr.NewClient()
// SDK handles the gRPC version internally
resp, err := client.GetState(ctx, "statestore", "key", nil)
```

## SDK Alignment with API Versions

SDK versions track Dapr runtime versions. The Go SDK minor version generally aligns with the Dapr runtime minor version:

```bash
# Check SDK version in go.mod
grep "github.com/dapr/go-sdk" go.mod

# Output: github.com/dapr/go-sdk v1.11.0
# Corresponds to Dapr runtime 1.11.x
```

## Checking Supported APIs at Runtime

Query the Dapr metadata endpoint to see which features are available:

```bash
curl http://localhost:3500/v1.0/metadata | jq '{
  runtimeVersion: .runtimeMetadata.runtimeVersion,
  enabledFeatures: .runtimeMetadata.enabledFeatures
}'
```

## Handling API Version Changes in Code

Wrap API calls with version constants to simplify future migrations:

```go
const DaprBaseURL = "http://localhost:3500"
const DaprAPIVersion = "v1.0"

func getState(storeName, key string) (*http.Response, error) {
    url := fmt.Sprintf("%s/%s/state/%s/%s",
        DaprBaseURL, DaprAPIVersion, storeName, key)
    return http.Get(url)
}
```

## Summary

Dapr uses URL-based API versioning with `v1.0` for stable APIs and `v1.0-alpha1` for preview features. Alpha APIs can change or be removed, so avoid them in production without a migration plan. Using the official Dapr SDKs abstracts versioning details and ensures compatibility as you upgrade the Dapr runtime.
