# How to Use Portainer Environment Variables for Secrets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Environment Variables, Secrets, Docker, Security, DevOps

Description: Learn how to manage environment variables and secrets in Portainer stacks, including best practices for keeping sensitive values out of your Compose files.

---

Environment variables are the most common way to pass configuration to containers. Portainer provides several mechanisms for managing them — from inline Compose definitions to Portainer's own stack environment variable store. This guide covers best practices for using environment variables for secrets without hardcoding sensitive values.

---

## The Problem with Hardcoded Secrets

Never put secrets directly in your Compose files. They'll end up in version control.

```yaml
# BAD: secrets hardcoded in Compose file
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: mysecretpassword  # visible in git history forever
```

---

## Option 1: Portainer Stack Environment Variables

Portainer lets you define environment variables in the stack editor that are stored securely and injected at deployment time.

In Portainer's Stack editor:
1. Open the **Environment variables** section
2. Add key/value pairs (e.g., `DB_PASSWORD` → `mysecretpassword`)
3. Reference them in the Compose file with `${DB_PASSWORD}`

```yaml
# Compose file using Portainer-managed env vars
version: "3.8"

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}      # resolved from Portainer's env store
      POSTGRES_USER: ${DB_USER}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

The values are stored in Portainer's database (not in the Compose file) and are not exposed in the UI after saving.

---

## Option 2: .env Files with Portainer

For stacks deployed from Git repositories, Portainer can load a `.env` file from the repository root.

```bash
# .env file — committed without real values
DB_PASSWORD=changeme
DB_USER=appuser
DB_NAME=myapp
API_KEY=your-api-key-here

# .env.production — NOT committed, contains real values
DB_PASSWORD=production-super-secret-password
DB_USER=produser
DB_NAME=prodapp
API_KEY=real-production-api-key
```

In Portainer stack settings with Git source, upload your `.env.production` file as the environment override.

---

## Option 3: Docker Swarm Secrets (Most Secure)

For Swarm deployments, use Docker secrets instead of environment variables.

```yaml
# swarm-stack.yml — using secrets instead of env vars
version: "3.8"

services:
  app:
    image: myapp:latest
    secrets:
      - db_password
    environment:
      # Reference the secret file path (never the value directly)
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true   # created separately via: echo "value" | docker secret create db_password -
```

---

## Option 4: Portainer API for Programmatic Secret Injection

Use the Portainer API to update stack environment variables from a CI/CD pipeline without touching the Compose file.

```bash
# Update stack environment variables via Portainer API
PORTAINER_URL="https://portainer.example.com"
API_KEY="your-portainer-api-key"
STACK_ID="5"

# Get current stack file
STACK=$(curl -s -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/stacks/$STACK_ID")

# Update the stack with new env vars
curl -X PUT \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"StackFileContent\": $(echo $STACK | jq '.StackFileContent'),
    \"Env\": [
      {\"name\": \"DB_PASSWORD\", \"value\": \"new-secure-password\"},
      {\"name\": \"API_KEY\", \"value\": \"new-api-key\"}
    ],
    \"Prune\": false
  }" \
  "$PORTAINER_URL/api/stacks/$STACK_ID"
```

---

## Best Practices Summary

| Approach | Use Case | Security Level |
|---|---|---|
| Hardcoded in Compose | Never | Low |
| Portainer env var store | Small teams, simple setups | Medium |
| `.env` file (not committed) | Git-deployed stacks | Medium |
| Docker Swarm secrets | Production Swarm workloads | High |
| External secrets manager (Vault, Infisical) | Enterprise, compliance | Very High |

---

## Summary

Portainer's environment variable store is the easiest way to keep secrets out of your Compose files for small deployments. For production Swarm workloads, Docker secrets provide the strongest protection by mounting values as files rather than environment variables. Always pair any approach with a `.gitignore` entry for `.env` files containing real credentials.
