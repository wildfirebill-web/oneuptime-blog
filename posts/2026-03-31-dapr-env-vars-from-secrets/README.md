# How to Use Environment Variables from Secrets in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Secret, Environment Variable, Kubernetes, Security

Description: Learn how to inject secrets as environment variables into Dapr-enabled services using Kubernetes secrets and Dapr's secret store integration.

---

## Overview

Exposing secrets as environment variables is a common pattern in containerized applications. Dapr enhances this workflow by providing a secret store API and supporting Kubernetes-native secret injection, giving you flexibility in how secrets reach your application.

## Method 1: Native Kubernetes Secret Injection

The simplest approach uses standard Kubernetes secret references in your pod spec:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded
  API_KEY: bXlzZWNyZXRrZXk=
```

Reference these in your Deployment:

```yaml
spec:
  containers:
  - name: myapp
    image: myrepo/myapp:latest
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: API_KEY
```

## Method 2: Fetching Secrets at Startup via Dapr

For dynamic secret retrieval at runtime, use the Dapr secret API at application startup:

```python
import os
from dapr.clients import DaprClient

def load_secrets_as_env():
    with DaprClient() as client:
        db_secret = client.get_secret(
            store_name="vault",
            key="db-credentials"
        )
        # Set as environment variables for the process
        os.environ["DB_HOST"] = db_secret.secret["host"]
        os.environ["DB_PASSWORD"] = db_secret.secret["password"]

        api_secret = client.get_secret(
            store_name="vault",
            key="api-keys"
        )
        os.environ["STRIPE_KEY"] = api_secret.secret["stripe"]

if __name__ == "__main__":
    load_secrets_as_env()
    # Now start the application
    run_app()
```

## Method 3: Init Container Pattern

Use an init container to fetch secrets and write them to a shared volume:

```yaml
initContainers:
- name: secret-loader
  image: dapr/dapr:latest
  command: ["/bin/sh", "-c"]
  args:
  - |
    curl http://localhost:3500/v1.0/secrets/vault/db-credentials \
      -o /secrets/db.json
  volumeMounts:
  - name: secrets
    mountPath: /secrets
```

Then read in the main container:

```python
import json

with open("/secrets/db.json") as f:
    db_creds = json.load(f)
db_password = db_creds["password"]
```

## Restricting Secret Access via Scoping

Use Dapr Configuration to limit which secrets each service can access:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: payment-config
spec:
  secrets:
    scopes:
    - storeName: vault
      defaultAccess: deny
      allowedSecrets:
      - stripe-keys
      - db-credentials
```

## Auditing Secret Access

Enable Dapr logging to track secret retrievals:

```bash
kubectl logs -l app=payment-service -c daprd | grep "secret"
```

## Summary

Dapr provides multiple approaches for getting secrets into environment variables: Kubernetes native injection for static values, the Dapr secret API for dynamic retrieval at runtime, and init containers for pre-startup loading. Combine with Configuration-level scoping to enforce least-privilege access and audit which services are accessing which secrets.
