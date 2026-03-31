# How to Configure Dapr with Local File Secret Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Secret Management, Local File, Local Development, Configuration

Description: Learn how to configure the Dapr local file secret store for development and testing, including nested secrets and multi-value secrets support.

---

The Dapr local file secret store reads secrets from a JSON file on disk. It supports single-level and nested key structures, making it flexible for storing structured secrets like database connection objects or grouped API credentials. This store is ideal for local development and CI environments where you want to simulate a real secret store without external dependencies.

## JSON File Format

Create a JSON file with your secrets. Single-level keys are the simplest:

```json
{
  "db-password": "localpassword",
  "api-key": "test-api-key-123",
  "stripe-secret": "sk_test_local",
  "redis-password": "localredispass"
}
```

For grouped secrets (multi-value secrets), use nested objects:

```json
{
  "database": {
    "password": "localpassword",
    "username": "devuser",
    "host": "localhost",
    "port": "5432"
  },
  "external-apis": {
    "stripe-key": "sk_test_local",
    "sendgrid-key": "SG.test.local"
  }
}
```

## Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: local-file-secrets
spec:
  type: secretstores.local.file
  version: v1
  metadata:
    - name: secretsFile
      value: "/home/user/.dapr/secrets.json"
    - name: nestedSeparator
      value: ":"
    - name: multiValued
      value: "false"
```

The `nestedSeparator` setting defines how nested keys are accessed. With `":"` as the separator, the database username is accessed as `database:username`.

## Retrieving Secrets

Single-level secret:

```bash
curl http://localhost:3500/v1.0/secrets/local-file-secrets/db-password
# Returns: {"db-password":"localpassword"}
```

Nested secret with separator:

```bash
curl http://localhost:3500/v1.0/secrets/local-file-secrets/database:password
# Returns: {"database:password":"localpassword"}
```

Multi-value secret (entire nested object):

```bash
curl http://localhost:3500/v1.0/secrets/local-file-secrets/database
```

When `multiValued` is `true`:

```json
{
  "password": "localpassword",
  "username": "devuser",
  "host": "localhost",
  "port": "5432"
}
```

## Organizing the Secrets File

For larger projects, structure the file to mirror your production secret store layout:

```json
{
  "postgres-main": {
    "username": "appuser",
    "password": "devpass",
    "host": "localhost:5432",
    "database": "myapp_dev"
  },
  "redis-cache": {
    "password": "redispass",
    "host": "localhost:6379"
  },
  "stripe": {
    "publishable-key": "pk_test_abc",
    "secret-key": "sk_test_abc",
    "webhook-secret": "whsec_test"
  }
}
```

## .gitignore the Secrets File

Never commit your local secrets file:

```bash
echo "secrets.json" >> .gitignore
echo ".dapr/secrets.json" >> .gitignore
```

Provide a `secrets.json.example` file with placeholder values for new developers:

```json
{
  "db-password": "REPLACE_WITH_YOUR_PASSWORD",
  "api-key": "REPLACE_WITH_YOUR_API_KEY"
}
```

## Summary

The Dapr local file secret store provides a file-backed secrets backend that supports both flat and nested key structures through configurable separators and multi-value modes. It is ideal for local development workflows where you want to use the same Dapr secrets API as production but without running an external secret store service.
