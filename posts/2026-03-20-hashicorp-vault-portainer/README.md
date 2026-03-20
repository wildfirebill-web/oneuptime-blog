# How to Integrate HashiCorp Vault with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, HashiCorp Vault, Secrets, Security, Docker, DevOps

Description: Learn how to deploy HashiCorp Vault via Portainer and integrate it with your containerized applications for dynamic secret management.

---

HashiCorp Vault is the gold standard for secrets management in containerized environments. It provides dynamic secrets (credentials generated on demand), secret rotation, audit logging, and fine-grained access policies. This guide shows how to deploy Vault as a Portainer stack and integrate it with your container workloads.

---

## Step 1: Deploy Vault via Portainer Stack

Start with a development Vault instance to learn the workflow, then harden it for production.

```yaml
# vault-stack.yml - HashiCorp Vault deployment

version: "3.8"

services:
  vault:
    image: hashicorp/vault:latest
    container_name: vault
    restart: unless-stopped
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "root-dev-token"    # dev mode only - change for production
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
      VAULT_ADDR: "http://0.0.0.0:8200"
    cap_add:
      - IPC_LOCK    # required for memory locking (prevents secrets from swapping to disk)
    volumes:
      - vault_data:/vault/data
      - vault_config:/vault/config

volumes:
  vault_data:
  vault_config:
```

> Note: For production, use a proper Vault configuration file with TLS and a production storage backend (Raft, Consul, etc.).

---

## Step 2: Initialize and Configure Vault

```bash
# Set the Vault address
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="root-dev-token"

# Verify Vault is running
vault status

# Enable the KV secrets engine (version 2)
vault secrets enable -path=secret kv-v2

# Write a test secret
vault kv put secret/myapp/config \
  db_password="supersecretpassword" \
  api_key="myapp-api-key-12345"

# Read it back
vault kv get secret/myapp/config
```

---

## Step 3: Create a Policy for Your Application

Vault policies control what secrets each application can access.

```hcl
# myapp-policy.hcl - grant read-only access to myapp secrets
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# Deny access to other paths
path "secret/data/*" {
  capabilities = ["deny"]
}
```

```bash
# Apply the policy
vault policy write myapp-policy myapp-policy.hcl

# Create an AppRole for your application
vault auth enable approle

vault write auth/approle/role/myapp \
  secret_id_ttl=10m \
  token_num_uses=10 \
  token_ttl=20m \
  token_max_ttl=30m \
  policies=myapp-policy

# Get the role ID and generate a secret ID
vault read auth/approle/role/myapp/role-id
vault write -f auth/approle/role/myapp/secret-id
```

---

## Step 4: Use Vault Secrets in Portainer-Managed Containers

Use the Vault Agent sidecar to fetch and inject secrets into your application.

```yaml
# app-with-vault-stack.yml - app with Vault Agent sidecar
version: "3.8"

services:
  vault-agent:
    image: hashicorp/vault:latest
    restart: unless-stopped
    command: vault agent -config=/vault/config/agent.hcl
    environment:
      VAULT_ADDR: http://vault:8200
    volumes:
      - vault_agent_config:/vault/config
      - shared_secrets:/run/secrets   # write secrets here for the app to read
    depends_on:
      - vault

  myapp:
    image: myapp:latest
    restart: unless-stopped
    volumes:
      - shared_secrets:/run/secrets:ro   # read-only access to secrets
    depends_on:
      - vault-agent

volumes:
  vault_agent_config:
  shared_secrets:
```

---

## Step 5: Python Example - Fetch Secrets Directly from Vault

```python
# vault_secrets.py - fetch secrets using the hvac Python client
import hvac

def get_vault_secrets(vault_url: str, token: str, secret_path: str) -> dict:
    """Retrieve secrets from HashiCorp Vault."""
    client = hvac.Client(url=vault_url, token=token)

    # Verify authentication
    if not client.is_authenticated():
        raise Exception("Vault authentication failed")

    # Read secrets from KV v2
    response = client.secrets.kv.v2.read_secret_version(
        path=secret_path,
        mount_point="secret"
    )
    return response["data"]["data"]

# Usage inside a container
secrets = get_vault_secrets(
    vault_url="http://vault:8200",
    token="root-dev-token",
    secret_path="myapp/config"
)
db_password = secrets["db_password"]
```

---

## Summary

Deploying Vault via Portainer gives you a powerful secrets management layer for all your containerized workloads. The key workflow is: store secrets in Vault, create AppRole credentials for each application, and fetch secrets either via the Vault Agent sidecar or directly from the SDK. This approach eliminates secrets from Compose files and environment variables entirely.
