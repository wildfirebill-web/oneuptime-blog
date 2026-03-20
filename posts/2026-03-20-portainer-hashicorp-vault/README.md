# How to Integrate HashiCorp Vault with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, HashiCorp Vault, Secrets, Security, DevOps

Description: Integrate HashiCorp Vault with Portainer to provide dynamic secrets and centralized secrets management for containerized workloads.

## Introduction

HashiCorp Vault is an enterprise-grade secrets management platform that provides dynamic secrets, secret versioning, and fine-grained access control. Integrating Vault with Portainer allows containers to receive short-lived, automatically-rotated secrets instead of static credentials.

## Prerequisites

- Portainer managing Docker or Kubernetes environments
- HashiCorp Vault instance (or deploy via Portainer)
- Docker or Kubernetes workloads needing secrets

## Part 1: Deploy Vault via Portainer

```yaml
# vault-stack.yml - deploy as Portainer stack

version: '3.8'

services:
  vault:
    image: hashicorp/vault:1.15
    container_name: vault
    restart: unless-stopped
    ports:
      - "8200:8200"
    environment:
      VAULT_ADDR: http://0.0.0.0:8200
      VAULT_API_ADDR: http://vault:8200
    volumes:
      - vault-data:/vault/data
      - vault-config:/vault/config
      - vault-logs:/vault/logs
    command: vault server -config=/vault/config/config.hcl
    cap_add:
      - IPC_LOCK  # Prevent secrets from being swapped to disk

volumes:
  vault-data:
  vault-config:
  vault-logs:
```

```hcl
# vault-config.hcl - mount into vault-config volume
storage "file" {
  path = "/vault/data"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1  # Enable TLS in production!
}

api_addr = "http://vault:8200"
cluster_addr = "http://vault:8201"
ui = true
```

## Part 2: Initialize and Unseal Vault

```bash
# Initialize Vault
docker exec -e VAULT_ADDR=http://localhost:8200 vault \
  vault operator init -key-shares=5 -key-threshold=3

# Save the unseal keys and root token securely!
# Output:
# Unseal Key 1: xxx
# Unseal Key 2: xxx
# ...
# Root Token: hvs.xxx

# Unseal Vault (repeat 3 times with different keys)
docker exec -e VAULT_ADDR=http://localhost:8200 vault \
  vault operator unseal <UNSEAL_KEY_1>
docker exec -e VAULT_ADDR=http://localhost:8200 vault \
  vault operator unseal <UNSEAL_KEY_2>
docker exec -e VAULT_ADDR=http://localhost:8200 vault \
  vault operator unseal <UNSEAL_KEY_3>

# Login with root token
export VAULT_ADDR=http://vault-host:8200
export VAULT_TOKEN=hvs.YOUR_ROOT_TOKEN
vault status
```

## Part 3: Configure Vault for Container Secrets

```bash
# Enable KV secrets engine
vault secrets enable -path=portainer kv-v2

# Store secrets
vault kv put portainer/myapp \
  db_password="SecurePassword123!" \
  api_key="api-key-value" \
  jwt_secret="jwt-secret-value"

# Read secrets
vault kv get portainer/myapp

# Enable AppRole authentication for services
vault auth enable approle

# Create a policy for myapp
vault policy write myapp-policy - << 'EOF'
path "portainer/data/myapp" {
  capabilities = ["read"]
}
path "portainer/data/shared/*" {
  capabilities = ["read"]
}
EOF

# Create an AppRole for myapp
vault write auth/approle/role/myapp \
  token_policies="myapp-policy" \
  token_ttl=1h \
  token_max_ttl=4h

# Get Role ID and Secret ID
ROLE_ID=$(vault read -field=role_id auth/approle/role/myapp/role-id)
SECRET_ID=$(vault write -force -field=secret_id auth/approle/role/myapp/secret-id)

echo "Role ID: $ROLE_ID"
echo "Secret ID: $SECRET_ID"
```

## Part 4: Using Vault Agent in Containers

```yaml
# app-with-vault-stack.yml
version: '3.8'

services:
  vault-agent:
    image: hashicorp/vault:1.15
    restart: unless-stopped
    environment:
      VAULT_ADDR: http://vault:8200
      ROLE_ID: "${ROLE_ID}"
      SECRET_ID: "${SECRET_ID}"
    volumes:
      - vault-agent-config:/vault/config
      - app-secrets:/vault/secrets  # Shared with app container
    command: vault agent -config=/vault/config/agent.hcl
    
  app:
    image: myapp:latest
    restart: unless-stopped
    depends_on:
      - vault-agent
    volumes:
      - app-secrets:/run/secrets  # Read secrets from here
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
      API_KEY_FILE: /run/secrets/api_key

volumes:
  vault-agent-config:
  app-secrets:
```

```hcl
# vault-agent config (agent.hcl)
vault {
  address = "http://vault:8200"
}

auto_auth {
  method "approle" {
    config = {
      role_id_file_path   = "/vault/config/role_id"
      secret_id_file_path = "/vault/config/secret_id"
    }
  }
  sink "file" {
    config = {
      path = "/vault/secrets/.vault_token"
    }
  }
}

template {
  source      = "/vault/config/db_password.tpl"
  destination = "/vault/secrets/db_password"
  perms       = 0640
}

template {
  source      = "/vault/config/api_key.tpl"
  destination = "/vault/secrets/api_key"
  perms       = 0640
}
```

## Part 5: Kubernetes Integration (Vault Injector)

```bash
# Install Vault Helm chart on Kubernetes
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace

# Enable Kubernetes auth
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://$(kubectl get svc kubernetes -o jsonpath='{.spec.clusterIP}')"

# Annotate pods to inject secrets
kubectl annotate pod mypod \
  vault.hashicorp.com/agent-inject=true \
  vault.hashicorp.com/role=myapp \
  vault.hashicorp.com/agent-inject-secret-config=portainer/myapp
```

## Conclusion

HashiCorp Vault integration with Portainer provides enterprise-grade secrets management with dynamic credentials, automatic rotation, and fine-grained access control. Vault Agent handles secret lifecycle automatically, ensuring containers always have fresh credentials without redeployment. For Kubernetes deployments, the Vault Injector provides seamless secret injection via pod annotations.
