# How to Implement API Key Authentication with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, API Key, Authentication, Security, Middleware

Description: Protect Dapr service invocation endpoints with API key authentication using the built-in middleware component and Kubernetes secrets.

---

## API Key Auth with Dapr Middleware

Dapr's `middleware.http.apikey` component validates API keys on incoming requests. The key is stored in a Kubernetes Secret and referenced by the component, keeping it out of your code and manifests.

## Creating the API Key Secret

```bash
# Generate a secure API key
API_KEY=$(openssl rand -hex 32)
echo "Generated key: $API_KEY"

# Store in Kubernetes secret
kubectl create secret generic api-key-secret \
  --from-literal=key="$API_KEY" \
  -n default
```

## API Key Middleware Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: api-key-validator
  namespace: default
spec:
  type: middleware.http.apikey
  version: v1
  metadata:
  - name: headerName
    value: "X-API-Key"
  - name: apiKey
    secretKeyRef:
      name: api-key-secret
      key: key
```

## Applying to the HTTP Pipeline

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: secured-api-config
  namespace: default
spec:
  httpPipeline:
    handlers:
    - name: api-key-validator
      type: middleware.http.apikey
```

## Multi-Key Validation (Custom Middleware)

For multiple API keys (e.g., one per client), use the Dapr Secrets API:

```go
package main

import (
    "context"
    "net/http"

    dapr "github.com/dapr/go-sdk/client"
)

var daprClient dapr.Client

func init() {
    var err error
    daprClient, err = dapr.NewClient()
    if err != nil {
        panic(err)
    }
}

func apiKeyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        incomingKey := r.Header.Get("X-API-Key")
        if incomingKey == "" {
            http.Error(w, `{"error":"missing API key"}`, http.StatusUnauthorized)
            return
        }

        // Look up key in secrets store
        secret, err := daprClient.GetSecret(context.Background(),
            "kubernetes", "api-keys", map[string]string{"namespace": "default"})
        if err != nil || secret[incomingKey] == "" {
            http.Error(w, `{"error":"invalid API key"}`, http.StatusForbidden)
            return
        }

        // Attach client ID to request context
        clientID := secret[incomingKey]
        r.Header.Set("X-Client-ID", clientID)
        next.ServeHTTP(w, r)
    })
}
```

## Kubernetes Secret with Multiple Keys

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  namespace: default
type: Opaque
stringData:
  # Format: key-value where key=API key, value=client identifier
  "abc123def456": "client-partner-a"
  "xyz789uvw012": "client-partner-b"
  "qrs345tuv678": "client-internal"
```

## Testing API Key Auth

```bash
# Without API key - rejected
curl http://localhost:3500/v1.0/invoke/api-service/method/data
# 401 Unauthorized

# With valid API key - accepted
curl -H "X-API-Key: $API_KEY" \
  http://localhost:3500/v1.0/invoke/api-service/method/data
# 200 OK

# Rotate API key without downtime (update secret, both old and new keys valid during rotation)
kubectl patch secret api-key-secret \
  -p '{"stringData":{"key":"new-key-value"}}'
```

## Summary

Dapr's API key middleware provides a zero-code authentication layer for service invocation endpoints. Storing keys in Kubernetes Secrets separates credentials from configuration, and the sidecar handles all validation before requests reach your application. For multi-client scenarios, the Secrets API enables per-client key lookup with client identity forwarding.
