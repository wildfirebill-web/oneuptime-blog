# How to Deploy Vault (HashiCorp) via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Vault, HashiCorp, Secrets Management, Docker, Security

Description: Learn how to deploy HashiCorp Vault via Portainer for centralized secrets management, with a file storage backend suitable for small teams and homelab use.

---

HashiCorp Vault securely stores and controls access to tokens, passwords, certificates, and API keys. Deploying it via Portainer lets you manage the container lifecycle and inspect logs without SSH access.

## Prerequisites

- Portainer running
- At least 256MB RAM
- Basic understanding of Vault unseal keys

## Compose Stack

This stack runs Vault in development mode for testing, and includes a production configuration example using the file storage backend:

```yaml
version: "3.8"

services:
  vault:
    image: hashicorp/vault:latest
    restart: unless-stopped
    ports:
      - "8200:8200"
    cap_add:
      - IPC_LOCK     # Required: prevents Vault memory from being swapped
    environment:
      VAULT_ADDR: http://0.0.0.0:8200
      VAULT_API_ADDR: http://0.0.0.0:8200
    volumes:
      - vault_data:/vault/file
      - vault_config:/vault/config
    # Run Vault with the file storage backend
    command: vault server -config=/vault/config/config.hcl

volumes:
  vault_data:
  vault_config:
```

## Vault Configuration File

Before deploying, write the config file into the `vault_config` volume. You can do this with an init container or by exec-ing into the container after first start:

```hcl
# /vault/config/config.hcl
storage "file" {
  path = "/vault/file"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1           # Enable TLS in production
}

ui = true                   # Enable the web UI
api_addr = "http://0.0.0.0:8200"
```

## Initializing Vault

After the container starts, initialize Vault via the CLI or UI:

```bash
# Export the Vault address
export VAULT_ADDR=http://localhost:8200

# Initialize with 5 key shares, threshold of 3 to unseal
vault operator init -key-shares=5 -key-threshold=3

# Unseal using three of the five unseal keys
vault operator unseal <unseal-key-1>
vault operator unseal <unseal-key-2>
vault operator unseal <unseal-key-3>

# Log in with the root token from the init output
vault login <root-token>
```

Store unseal keys and the root token securely offline. Vault must be unsealed after every restart.

## Writing and Reading a Secret

```bash
# Enable the KV secrets engine
vault secrets enable -path=secret kv-v2

# Write a secret
vault kv put secret/myapp db_password="supersecret"

# Read it back
vault kv get secret/myapp
```

## Monitoring

Use OneUptime to monitor `http://<host>:8200/v1/sys/health`. Vault returns `200` when initialized and unsealed, `503` when sealed, and `501` when uninitialized. Alert on anything other than `200` to catch an accidentally sealed Vault before your applications do.
