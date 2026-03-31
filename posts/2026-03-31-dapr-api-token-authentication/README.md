# How to Use API Token Authentication for Dapr APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Security, Authentication, API, Token

Description: Learn how to enable API token authentication in Dapr to protect your Dapr HTTP and gRPC APIs from unauthorized access using a shared secret token.

---

## Overview

By default, the Dapr sidecar exposes its APIs to any process that can reach it on the local network. API token authentication adds a layer of protection so that only callers presenting the correct token can invoke Dapr APIs. This is especially important when running in shared or multi-tenant environments.

## Enabling API Token Authentication

You enable API token authentication by setting the `DAPR_API_TOKEN` environment variable on the Dapr sidecar. Any incoming request must include this token in the `dapr-api-token` HTTP header.

### Kubernetes Setup

Create a Kubernetes secret for the token:

```bash
kubectl create secret generic dapr-api-token \
  --from-literal=token=supersecrettoken123
```

Reference the secret in your pod's Dapr annotations:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "my-service"
  dapr.io/api-token-secret: "dapr-api-token"
```

The `dapr.io/api-token-secret` annotation tells the injector to set `DAPR_API_TOKEN` from the named secret.

### Self-Hosted Setup

Set the environment variable before running the Dapr process:

```bash
export DAPR_API_TOKEN="supersecrettoken123"
dapr run --app-id my-service --app-port 3000 -- node app.js
```

## Calling Dapr APIs with the Token

Every call to the Dapr sidecar must now include the token header:

```bash
curl -X POST http://localhost:3500/v1.0/state/my-store \
  -H "Content-Type: application/json" \
  -H "dapr-api-token: supersecrettoken123" \
  -d '[{"key":"user1","value":"Alice"}]'
```

In application code (Node.js example):

```javascript
const axios = require('axios');

async function saveState(key, value) {
  await axios.post('http://localhost:3500/v1.0/state/my-store',
    [{ key, value }],
    {
      headers: {
        'dapr-api-token': process.env.DAPR_API_TOKEN,
        'Content-Type': 'application/json'
      }
    }
  );
}
```

## Using the Dapr SDK

The official Dapr SDKs support API tokens natively. Set the `DAPR_API_TOKEN` environment variable and the SDK picks it up automatically:

```bash
export DAPR_API_TOKEN="supersecrettoken123"
```

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();
// Token is automatically included in all requests
await client.state.save('my-store', [{ key: 'user1', value: 'Alice' }]);
```

## Verifying Token Rejection

Test that requests without the token are rejected:

```bash
curl -X GET http://localhost:3500/v1.0/metadata
# Expected: 401 Unauthorized
```

## Summary

Dapr API token authentication is a simple but effective way to prevent unauthorized access to the Dapr sidecar. By setting a secret token via environment variable and passing it in the `dapr-api-token` header, you add a critical security layer to your microservices deployment with minimal code changes.
