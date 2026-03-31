# How to Get Bulk Secrets from a Dapr Secret Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Secret, Bulk Secret, Secret Store, Configuration

Description: Use the Dapr bulk secrets API to retrieve all secrets from a store in a single call, useful for loading application configuration at startup.

---

The Dapr bulk secrets endpoint retrieves all secrets accessible in a secret store with a single API call. This is useful for loading all application configuration secrets at startup rather than making individual calls for each key.

## Bulk Secrets HTTP API

```text
GET http://localhost:{daprPort}/v1.0/secrets/{storeName}/bulk
```

Example:

```bash
curl http://localhost:3500/v1.0/secrets/mysecretstore/bulk
```

Response:

```json
{
  "database-password": {
    "database-password": "supersecure123"
  },
  "api-key": {
    "api-key": "sk-prod-abc123xyz"
  },
  "jwt-secret": {
    "jwt-secret": "my-jwt-signing-secret"
  },
  "smtp-credentials": {
    "username": "smtp-user@example.com",
    "password": "smtp-password-here"
  }
}
```

Note: The response is a map of secret names to their key-value maps. Each entry can have multiple fields for backends that support compound secrets.

## Bulk Secrets in Go

```go
package main

import (
  "context"
  "log"
  dapr "github.com/dapr/go-sdk/client"
)

type AppConfig struct {
  DatabasePassword string
  APIKey           string
  JWTSecret        string
  SMTPUser         string
  SMTPPassword     string
}

func loadConfig(ctx context.Context) (*AppConfig, error) {
  client, err := dapr.NewClient()
  if err != nil {
    return nil, err
  }
  defer client.Close()

  secrets, err := client.GetBulkSecret(ctx, "mysecretstore", nil)
  if err != nil {
    return nil, fmt.Errorf("failed to load secrets: %w", err)
  }

  config := &AppConfig{}

  if dbSecret, ok := secrets["database-password"]; ok {
    config.DatabasePassword = dbSecret["database-password"]
  }
  if apiSecret, ok := secrets["api-key"]; ok {
    config.APIKey = apiSecret["api-key"]
  }
  if jwtSecret, ok := secrets["jwt-secret"]; ok {
    config.JWTSecret = jwtSecret["jwt-secret"]
  }
  if smtp, ok := secrets["smtp-credentials"]; ok {
    config.SMTPUser = smtp["username"]
    config.SMTPPassword = smtp["password"]
  }

  return config, nil
}
```

## Bulk Secrets in Python

```python
from dapr.clients import DaprClient

def load_all_config() -> dict:
    with DaprClient() as client:
        secrets = client.get_bulk_secret(store_name="mysecretstore")

        config = {}
        for name, kv_map in secrets.secrets.items():
            for key, value in kv_map.items():
                config[f"{name}.{key}"] = value

        return config

config = load_all_config()
print(f"Loaded {len(config)} secret values")
```

## Using Metadata for Prefix Filtering

Some backends support filtering secrets by prefix. With AWS Secrets Manager:

```bash
curl "http://localhost:3500/v1.0/secrets/aws-store/bulk?metadata.path=myapp/prod"
```

This returns only secrets under the `myapp/prod/` path.

## Caching Bulk Secrets at Startup

```go
var (
  configOnce sync.Once
  appConfig  *AppConfig
)

func GetConfig(ctx context.Context) *AppConfig {
  configOnce.Do(func() {
    var err error
    appConfig, err = loadConfig(ctx)
    if err != nil {
      log.Fatalf("Failed to load application config: %v", err)
    }
    log.Printf("Loaded %d secret values from store", countFields(appConfig))
  })
  return appConfig
}
```

## Access Control Considerations

The bulk secrets API returns all secrets your app is authorized to see based on scoping rules. If your component has `allowedSecrets` configured, the bulk endpoint only returns those secrets:

```yaml
spec:
  type: secretstores.kubernetes
  version: v1
  allowedSecrets:
  - database-password
  - api-key
  - jwt-secret
```

## Summary

The Dapr bulk secrets API provides an efficient way to load all application configuration secrets at startup with a single request. This pattern reduces the number of secret store API calls compared to fetching secrets individually, and simplifies the initialization code for config-heavy applications. Combine bulk loading with in-memory caching to avoid repeated secret store calls during normal operation.
