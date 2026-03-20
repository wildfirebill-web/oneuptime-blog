# How to Manage Docker Secrets in Portainer on Swarm - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Secrets, Security, DevOps

Description: Learn how to securely create, manage, and attach Docker Swarm secrets to services using Portainer for secrets management.

## Introduction

Docker Secrets provide a secure mechanism for distributing sensitive data - passwords, API keys, TLS certificates, and SSH keys - to Swarm services. Unlike environment variables, secrets are encrypted at rest in the Swarm raft log, transmitted over encrypted channels, and mounted as in-memory files in containers. Portainer provides a UI for managing secrets that is easier than the Docker CLI. This guide covers the complete secrets management workflow.

## Prerequisites

- Portainer on Docker Swarm
- Admin access to Portainer
- Sensitive data you need to distribute to services

## Understanding Docker Secrets Security Model

- Secrets are **encrypted at rest** in the Swarm raft log
- Only distributed to services that **explicitly request** them
- Mounted as files under `/run/secrets/<secret-name>` in the container
- **In-memory only** on worker nodes - not written to disk
- Secrets are **immutable** after creation

## Step 1: View Existing Secrets

1. Select your Swarm environment in Portainer
2. Click **Swarm → Secrets**

The list shows secret names and creation dates (but NOT the secret values - they are never shown after creation).

## Step 2: Create a New Secret

1. Click **+ Add secret**
2. Enter a **Secret name** (e.g., `db-password`)
3. Enter the secret **Value**:

```text
Secret name:  db-password
Value:        MyStr0ng!P@ssw0rd#2024
```

4. Click **Create secret**

**Important:** Once created, the secret value cannot be retrieved or viewed. Store it securely before creating.

## Step 3: Create Secrets via CLI

```bash
# Create from string (visible in shell history - avoid in production)

echo "MyStr0ng!P@ssw0rd" | docker secret create db-password -

# Create from file (preferred - avoids shell history exposure)
printf "MyStr0ng!P@ssw0rd" > /tmp/db-password.txt
docker secret create db-password /tmp/db-password.txt
rm /tmp/db-password.txt  # Clean up immediately

# Create from environment variable
printf "%s" "$DB_PASSWORD" | docker secret create db-password -

# List secrets
docker secret ls

# Inspect (shows metadata, not value)
docker secret inspect db-password
```

## Step 4: Attach Secrets to a Service

### Via Portainer Service Editor

1. Open a service for editing
2. Scroll to the **Secrets** section
3. Click **+ Add a secret**
4. Select the secret name
5. Configure mount options:

```text
Secret:       db-password
Target file:  db-password           (filename in /run/secrets/)
UID:          0                     (file owner UID)
GID:          0                     (file owner GID)
Mode:         0400                  (read-only for owner)
```

The secret will be available at `/run/secrets/db-password` inside the container.

### Via Stack Compose File

```yaml
version: "3.8"

services:
  api:
    image: myapi:latest
    secrets:
      - db-password           # Mount at /run/secrets/db-password
      - api-key               # Mount at /run/secrets/api-key
      - db-certificate        # TLS certificate
    environment:
      # Tell the app to read from the secret file
      - DB_PASSWORD_FILE=/run/secrets/db-password

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password  # postgres reads this
    secrets:
      - db-password

secrets:
  db-password:
    external: true     # Already created in the Swarm cluster
  api-key:
    external: true
  db-certificate:
    external: true
```

## Step 5: Read Secrets in Your Application

Applications must be coded to read secrets from files, not environment variables:

### Shell Script

```bash
#!/bin/sh
# Read secret from file
DB_PASSWORD=$(cat /run/secrets/db-password)
export DB_PASSWORD
```

### Python

```python
import os

def read_secret(secret_name):
    """Read a Docker secret from the filesystem"""
    secret_path = f"/run/secrets/{secret_name}"
    try:
        with open(secret_path, 'r') as f:
            return f.read().strip()
    except FileNotFoundError:
        # Fall back to environment variable for local development
        return os.environ.get(secret_name.upper().replace('-', '_'))

db_password = read_secret('db-password')
api_key = read_secret('api-key')
```

### Node.js

```javascript
const fs = require('fs');

function readSecret(secretName) {
    const secretPath = `/run/secrets/${secretName}`;
    try {
        return fs.readFileSync(secretPath, 'utf8').trim();
    } catch (err) {
        // Fall back to environment variable
        return process.env[secretName.toUpperCase().replace(/-/g, '_')];
    }
}

const dbPassword = readSecret('db-password');
```

## Step 6: Update a Secret

Like configs, secrets are immutable. To update:

```bash
# Create new secret version
printf "NewStr0ng!P@ssw0rd" | docker secret create db-password-v2 -

# Update service to use new secret
docker service update \
  --secret-rm db-password \
  --secret-add source=db-password-v2,target=db-password \
  my-api-service
```

## Step 7: Secret Rotation Strategy

```bash
#!/bin/bash
# rotate-secret.sh
SECRET_BASE_NAME="db-password"
NEW_VALUE="$1"  # Pass new value as argument

if [ -z "$NEW_VALUE" ]; then
    echo "Usage: $0 <new-secret-value>"
    exit 1
fi

TIMESTAMP=$(date +%Y%m%d%H%M)
NEW_SECRET_NAME="${SECRET_BASE_NAME}-${TIMESTAMP}"

# Create new secret
printf "%s" "$NEW_VALUE" | docker secret create "$NEW_SECRET_NAME" -

# Update all services using this secret
for svc in $(docker service ls -q); do
    if docker service inspect $svc --format '{{.Spec.TaskTemplate.ContainerSpec.Secrets}}' | grep -q "$SECRET_BASE_NAME"; then
        docker service update \
          --secret-rm "$SECRET_BASE_NAME" \
          --secret-add "source=${NEW_SECRET_NAME},target=${SECRET_BASE_NAME}" \
          $svc
        echo "Updated service $svc"
    fi
done
```

## Step 8: Remove Old Secrets

```bash
# Remove a secret (must not be in use by any service)
docker secret rm db-password-v1

# In Portainer: select and delete from the Secrets list
```

## Conclusion

Docker Secrets provide the most secure way to distribute sensitive data in Swarm environments. By storing secrets in Portainer and attaching them to services, you avoid embedding passwords and keys in environment variables, container images, or compose files. Always code your applications to read secrets from files, implement a rotation strategy for production credentials, and remove old secret versions promptly.
