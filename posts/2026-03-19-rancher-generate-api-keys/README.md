# How to Generate API Keys in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, API, REST API, Authentication

Description: Step-by-step guide to generating and managing API keys in Rancher for programmatic access to your Kubernetes clusters.

API keys are the foundation of programmatic access to Rancher. Whether you need to automate cluster provisioning, build custom integrations, or script routine maintenance tasks, you need properly configured API keys. This guide covers every method for creating and managing API keys in Rancher.

## Understanding Rancher API Key Types

Rancher supports two types of API keys:

- **Account API Keys**: Scoped to a specific user account. These keys inherit the permissions of the user who created them.
- **Scoped API Keys**: Limited to a specific cluster or project. These provide more granular access control.

Each API key consists of an Access Key (username) and a Secret Key (password), combined in the format `access_key:secret_key`.

## Generating API Keys Through the UI

The simplest way to create an API key is through the Rancher UI.

### Step 1: Navigate to API Keys

Log into your Rancher instance and click on your user avatar in the top-right corner. Select **Account & API Keys** from the dropdown menu.

### Step 2: Create a New API Key

Click the **Create API Key** button. You will see a form with the following fields:

- **Description**: A human-readable label for the key (e.g., "CI/CD Pipeline Key")
- **Scope**: Choose between "No Scope" (full access) or limit to a specific cluster
- **Expires**: Set an expiration time or leave blank for no expiration
- **Auto-delete expired key**: Toggle whether expired keys should be cleaned up automatically

### Step 3: Save Your Credentials

After clicking **Create**, Rancher displays the Access Key, Secret Key, and Bearer Token. Copy these immediately because the Secret Key and Bearer Token are only shown once.

```plaintext
Access Key:  token-abc12
Secret Key:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Bearer Token: token-abc12:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Store these securely in a password manager or secrets vault.

## Generating API Keys Through the API

You can also create API keys programmatically using an existing key or session token.

### Creating an Account-Scoped Key

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "token",
    "description": "Automation Key - Created via API",
    "ttl": 0
  }' \
  "${RANCHER_URL}/v3/tokens"
```

The `ttl` field is in milliseconds. Set it to `0` for a non-expiring key or use a value like `86400000` for a 24-hour key.

### Creating a Cluster-Scoped Key

To create a key scoped to a specific cluster:

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "token",
    "description": "Production Cluster Key",
    "ttl": 0,
    "clusterId": "c-m-abc12345"
  }' \
  "${RANCHER_URL}/v3/tokens"
```

### Parsing the Response

The response contains your new credentials:

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "token",
    "description": "Script Key"
  }' \
  "${RANCHER_URL}/v3/tokens" | jq '{
    accessKey: .token,
    secretKey: .token,
    bearerToken: "\(.token):\(.token)"
  }'
```

## Generating API Keys via the Rancher CLI

The Rancher CLI can also generate API keys:

```bash
rancher login https://rancher.example.com --token ${RANCHER_TOKEN}
rancher tokens create --description "CLI Generated Key" --ttl 0
```

## Managing Existing API Keys

### Listing All API Keys

To list all API keys for your account:

```bash
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/tokens" | jq '.data[] | {
    id: .id,
    description: .description,
    expired: .expired,
    created: .created,
    expiresAt: .expiresAt,
    clusterId: .clusterName
  }'
```

### Deleting an API Key

To revoke an API key:

```bash
TOKEN_ID="token-abc12"

curl -s -k -X DELETE \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/tokens/${TOKEN_ID}"
```

### Checking Key Validity

Test whether a key is still valid:

```bash
curl -s -k -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer token-abc12:xxxxxxxxxxxx" \
  "${RANCHER_URL}/v3/users?me=true"
```

A `200` response means the key is valid. A `401` means it has expired or been revoked.

## Best Practices for API Key Management

### Use Short-Lived Keys When Possible

For CI/CD pipelines that run on a schedule, create keys with a TTL that slightly exceeds the expected run time:

```bash
# Create a key that expires in 2 hours (7200000 ms)
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "token",
    "description": "CI Pipeline - Short-lived",
    "ttl": 7200000
  }' \
  "${RANCHER_URL}/v3/tokens"
```

### Use Scoped Keys for Least Privilege

Instead of granting full access, scope keys to specific clusters:

```bash
curl -s -k -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "token",
    "description": "Staging Only Key",
    "clusterId": "c-m-staging01"
  }' \
  "${RANCHER_URL}/v3/tokens"
```

### Rotate Keys Regularly

Build a rotation script that creates a new key and deletes the old one:

```bash
#!/bin/bash

RANCHER_URL="https://rancher.example.com"
OLD_TOKEN="token-old123:xxxxxxxxxxxx"

# Create new key using old key
NEW_KEY=$(curl -s -k -X POST \
  -H "Authorization: Bearer ${OLD_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "token",
    "description": "Rotated Key - '"$(date +%Y-%m-%d)"'"
  }' \
  "${RANCHER_URL}/v3/tokens")

NEW_TOKEN=$(echo "$NEW_KEY" | jq -r '.token')
echo "New token created: ${NEW_TOKEN}"

# Store the new token in your secrets manager
# vault kv put secret/rancher token="${NEW_TOKEN}"

# Delete old key
OLD_TOKEN_ID=$(echo "$OLD_TOKEN" | cut -d: -f1)
curl -s -k -X DELETE \
  -H "Authorization: Bearer ${NEW_TOKEN}" \
  "${RANCHER_URL}/v3/tokens/${OLD_TOKEN_ID}"

echo "Old token deleted: ${OLD_TOKEN_ID}"
```

### Store Keys Securely

Never store API keys in plain text files, environment variables in shared systems, or version control. Use a secrets manager:

```bash
# HashiCorp Vault
vault kv put secret/rancher/api-key token="${RANCHER_TOKEN}"

# AWS Secrets Manager
aws secretsmanager create-secret \
  --name rancher-api-key \
  --secret-string "${RANCHER_TOKEN}"

# Kubernetes Secret
kubectl create secret generic rancher-api-key \
  --from-literal=token="${RANCHER_TOKEN}" \
  -n automation
```

### Audit Key Usage

Periodically review all active keys and remove unused ones:

```bash
curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/tokens" | jq '.data[] | select(.expired == false) | {
    id,
    description,
    created,
    lastUsed: .lastUpdateTime
  }'
```

## Summary

Generating and managing API keys in Rancher is straightforward whether you use the UI, the API, or the CLI. The key practices to follow are using scoped keys with appropriate TTLs, rotating them regularly, and storing them in a secrets manager. With properly configured API keys, you can safely automate any Rancher operation.
