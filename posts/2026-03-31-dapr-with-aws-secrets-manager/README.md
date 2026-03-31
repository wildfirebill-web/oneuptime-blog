# How to Use Dapr with AWS Secrets Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, Secret, Security, Configuration

Description: Configure Dapr to retrieve secrets from AWS Secrets Manager, enabling services to access database passwords, API keys, and certificates without hardcoding credentials.

---

AWS Secrets Manager stores and rotates secrets securely. Dapr's secrets building block wraps Secrets Manager, letting services retrieve secrets through a uniform API and reference them in other Dapr components.

## Store Secrets in AWS Secrets Manager

```bash
# Create a database secret
aws secretsmanager create-secret \
  --name myapp/database \
  --description "Database credentials" \
  --secret-string '{"username":"dbuser","password":"s3cr3t","host":"db.example.com"}' \
  --region us-east-1

# Create an API key secret
aws secretsmanager create-secret \
  --name myapp/stripe-api-key \
  --secret-string '{"key":"sk_live_EXAMPLE123"}' \
  --region us-east-1
```

## Configure the Dapr Secrets Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: aws-secretstore
  namespace: default
spec:
  type: secretstores.aws.secretsmanager
  version: v1
  metadata:
  - name: region
    value: us-east-1
  - name: accessKey
    secretKeyRef:
      name: aws-bootstrap-credentials
      key: accessKey
  - name: secretKey
    secretKeyRef:
      name: aws-bootstrap-credentials
      key: secretKey
```

## Retrieve a Secret in Your Application

```python
import requests

def get_secret(secret_name: str, key: str = None) -> dict:
    url = f"http://localhost:3500/v1.0/secrets/aws-secretstore/{secret_name}"
    resp = requests.get(url)
    resp.raise_for_status()
    secrets = resp.json()
    if key:
        return secrets.get(key)
    return secrets

# Get all fields from a secret
db_creds = get_secret("myapp/database")
print(f"Host: {db_creds['host']}")
print(f"User: {db_creds['username']}")

# Get a specific key
api_key = get_secret("myapp/stripe-api-key", "key")
print(f"API Key: {api_key[:8]}...")
```

## Reference Secrets in Other Dapr Components

Use secrets from Secrets Manager in Dapr component definitions:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  auth:
    secretStore: aws-secretstore
  metadata:
  - name: redisHost
    value: redis:6379
  - name: redisPassword
    secretKeyRef:
      name: myapp/redis
      key: password
```

## Bulk Secret Retrieval

```python
def get_all_secrets() -> dict:
    resp = requests.get(
        "http://localhost:3500/v1.0/secrets/aws-secretstore/bulk"
    )
    resp.raise_for_status()
    return resp.json()

secrets = get_all_secrets()
for name, values in secrets.items():
    print(f"Secret: {name}, Keys: {list(values.keys())}")
```

## Restrict Access with Scopes

Limit which secrets each application can access:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: aws-secretstore
  namespace: default
spec:
  type: secretstores.aws.secretsmanager
  version: v1
  metadata:
  - name: region
    value: us-east-1
scopes:
- order-service
- payment-service
```

## IAM Policy for Secrets Manager

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/*"
    }
  ]
}
```

## Summary

Dapr's AWS Secrets Manager integration centralizes secret access through the `/v1.0/secrets` API, eliminating hardcoded credentials in application code. Secrets can be referenced directly in other Dapr component definitions, and component scopes limit which applications can access which secrets. Combined with IRSA for authentication, this provides a fully credential-free deployment pipeline.
