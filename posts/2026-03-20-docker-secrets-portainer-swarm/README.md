# How to Manage Docker Secrets via Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Secrets, Security, Docker, DevOps

Description: Learn how to create and manage Docker Swarm secrets through Portainer's UI and use them securely in your Swarm service stacks.

---

Docker Swarm secrets let you store sensitive data - passwords, API keys, certificates - in the cluster's encrypted store and mount them into containers as files. Portainer provides a UI for managing these secrets across your Swarm without exposing values in Compose files or environment variables.

---

## How Docker Secrets Work

Secrets are stored encrypted in Raft (the Swarm consensus store) and mounted into containers at `/run/secrets/<secret_name>`. Only services explicitly granted access to a secret can read it. The values are never visible in `docker inspect`, environment listings, or Portainer's UI once created.

---

## Step 1: Create Secrets via Portainer UI

1. In Portainer, select your Swarm environment
2. Navigate to **Swarm > Secrets**
3. Click **Add secret**
4. Enter:
   - **Name**: `db_password`
   - **Value**: `my-super-secret-password`
5. Click **Create the secret**

Alternatively, use the CLI:

```bash
# Create a secret from a string

echo "my-super-secret-password" | docker secret create db_password -

# Create a secret from a file (e.g., a TLS certificate)
docker secret create ssl_cert /path/to/cert.pem

# List all secrets (values are never shown)
docker secret ls
```

---

## Step 2: Use Secrets in a Swarm Stack

Reference secrets in your Docker Compose file using the `secrets:` key.

```yaml
# app-with-secrets-stack.yml - Swarm stack using Docker secrets
version: "3.8"

services:
  web:
    image: myapp:latest
    restart: unless-stopped
    # Grant this service access to specific secrets
    secrets:
      - db_password
      - api_key
    environment:
      # Point the app to the secret file path
      DB_PASSWORD_FILE: /run/secrets/db_password
      API_KEY_FILE: /run/secrets/api_key
    deploy:
      replicas: 3

  db:
    image: postgres:15
    restart: unless-stopped
    secrets:
      - db_password
    environment:
      # PostgreSQL supports _FILE suffix to read from a file
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser

# Reference the secrets created in the Swarm
secrets:
  db_password:
    external: true   # already exists in the Swarm
  api_key:
    external: true
```

---

## Step 3: Read Secret Values in Your Application Code

Your application reads secrets as plain text files from `/run/secrets/`.

```python
# Python example: reading a Docker secret
def read_secret(secret_name: str) -> str:
    """Read a Docker secret from the standard mount path."""
    secret_path = f"/run/secrets/{secret_name}"
    try:
        with open(secret_path, "r") as f:
            return f.read().strip()
    except FileNotFoundError:
        # Fall back to environment variable for local development
        import os
        return os.environ.get(secret_name.upper(), "")

# Usage
db_password = read_secret("db_password")
api_key = read_secret("api_key")
```

---

## Step 4: Rotate Secrets Safely

Docker secrets are immutable - you can't update a secret's value in place. To rotate a secret:

```bash
# Create a new version of the secret
echo "new-password-value" | docker secret create db_password_v2 -

# Update the stack to use the new secret name
# Edit the Compose file in Portainer to reference db_password_v2

# After deployment is stable, remove the old secret
docker secret rm db_password
```

---

## Step 5: View Secret Usage in Portainer

In Portainer:
- **Swarm > Secrets** - shows all secrets and which services use them
- **Swarm > Services** - each service shows which secrets are mounted
- Secret values are **never displayed** in the UI - only names and metadata

---

## Summary

Docker Swarm secrets managed via Portainer provide encrypted, access-controlled secret distribution across your cluster. Secrets mount as files at `/run/secrets/`, never appear in environment variable listings, and are only accessible to services you explicitly grant them to. Use the `_FILE` environment variable suffix for databases and other apps that support it natively.
