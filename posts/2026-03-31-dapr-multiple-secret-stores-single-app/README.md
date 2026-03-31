# How to Use Multiple Secret Stores in a Single Dapr Application

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Secret Stores, Security, Configuration, Kubernetes

Description: Learn how to configure and use multiple Dapr secret store components in a single application to read secrets from different backends like Vault and AWS Secrets Manager.

---

## Why Multiple Secret Stores

A single application may need secrets from different backends - for example, database credentials stored in HashiCorp Vault, cloud API keys in AWS Secrets Manager, and local development secrets in Kubernetes Secrets. Dapr allows you to define multiple secret store components and access each by name in your application.

## Configuring Multiple Secret Store Components

Define separate component files for each secret store:

```yaml
# components/vault-secrets.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault
  namespace: default
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
  - name: vaultAddr
    value: "http://vault.default.svc.cluster.local:8200"
  - name: skipVerify
    value: "false"
  - name: tlsServerName
    value: "vault"
  - name: vaultTokenMountPath
    value: "/var/run/secrets/vault/token"
```

```yaml
# components/aws-secrets.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: aws-secrets
  namespace: default
spec:
  type: secretstores.aws.secretmanager
  version: v1
  metadata:
  - name: region
    value: "us-east-1"
  - name: accessKey
    secretKeyRef:
      name: aws-credentials
      key: accessKeyId
  - name: secretKey
    secretKeyRef:
      name: aws-credentials
      key: secretAccessKey
```

```yaml
# components/k8s-secrets.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: k8s-secrets
  namespace: default
spec:
  type: secretstores.kubernetes
  version: v1
```

## Reading Secrets from Different Stores

Use the Dapr SDK to read from each store by name:

```python
from dapr.clients import DaprClient

def get_database_credentials():
    """Read database credentials from HashiCorp Vault."""
    with DaprClient() as client:
        secret = client.get_secret(
            store_name="vault",
            key="database/credentials"
        )
        return {
            "username": secret.secret.get("username"),
            "password": secret.secret.get("password"),
        }

def get_aws_api_key():
    """Read AWS API key from AWS Secrets Manager."""
    with DaprClient() as client:
        secret = client.get_secret(
            store_name="aws-secrets",
            key="myapp/aws-api-key"
        )
        return secret.secret.get("apiKey")

def get_local_config():
    """Read local config from Kubernetes Secrets."""
    with DaprClient() as client:
        secret = client.get_secret(
            store_name="k8s-secrets",
            key="app-local-config"
        )
        return secret.secret
```

## Centralizing Secret Access with a Helper

Create a secret manager class that abstracts which store a secret lives in:

```python
from dataclasses import dataclass
from dapr.clients import DaprClient

@dataclass
class SecretRef:
    store: str
    key: str
    field: str = None  # specific field within the secret

class SecretManager:
    SECRET_REGISTRY = {
        "db.host":        SecretRef(store="vault",       key="database/credentials", field="host"),
        "db.password":    SecretRef(store="vault",       key="database/credentials", field="password"),
        "stripe.api_key": SecretRef(store="aws-secrets", key="payments/stripe",      field="apiKey"),
        "jwt.secret":     SecretRef(store="k8s-secrets", key="auth-secrets",         field="jwtSecret"),
    }

    def __init__(self):
        self._cache = {}

    def get(self, name: str) -> str:
        if name in self._cache:
            return self._cache[name]

        ref = self.SECRET_REGISTRY.get(name)
        if not ref:
            raise KeyError(f"Unknown secret: {name}")

        with DaprClient() as client:
            secret = client.get_secret(store_name=ref.store, key=ref.key)
            value = secret.secret.get(ref.field) if ref.field else secret.secret
            self._cache[name] = value
            return value

# Usage
secrets = SecretManager()

db_password = secrets.get("db.password")
stripe_key = secrets.get("stripe.api_key")
jwt_secret = secrets.get("jwt.secret")
```

## Using the HTTP API

Access different stores via the Dapr sidecar HTTP API:

```bash
# Read from Vault
curl http://localhost:3500/v1.0/secrets/vault/database%2Fcredentials

# Read from AWS Secrets Manager
curl http://localhost:3500/v1.0/secrets/aws-secrets/payments%2Fstripe

# Read from Kubernetes Secrets
curl http://localhost:3500/v1.0/secrets/k8s-secrets/auth-secrets
```

## Bulk Secret Retrieval

Retrieve all secrets from a store at once:

```go
import (
    "context"
    "fmt"
    dapr "github.com/dapr/go-sdk/client"
)

func loadAllVaultSecrets(ctx context.Context, client dapr.Client) (map[string]map[string]string, error) {
    secrets, err := client.GetBulkSecret(ctx, "vault", nil)
    if err != nil {
        return nil, fmt.Errorf("failed to get bulk secrets from vault: %w", err)
    }

    fmt.Printf("Loaded %d secrets from Vault\n", len(secrets))
    return secrets, nil
}
```

## Access Control with Scopes

Restrict which apps can access which secret stores using Dapr component scopes:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault
  namespace: default
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
  - name: vaultAddr
    value: "http://vault:8200"
scopes:
  - payment-service    # Only payment-service can access Vault
  - order-service
```

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: aws-secrets
  namespace: default
spec:
  type: secretstores.aws.secretmanager
  version: v1
  metadata:
  - name: region
    value: "us-east-1"
scopes:
  - analytics-service  # Only analytics-service can access AWS secrets
```

## Summary

Dapr makes it straightforward to work with multiple secret backends in a single application by treating each secret store as a named component. Define separate YAML files for each backend, read from them by name in your code, and use scopes to enforce which services can access which stores. A centralized secret registry pattern reduces hardcoded store names throughout your codebase and makes it easy to change which backend holds a secret without modifying application code.
