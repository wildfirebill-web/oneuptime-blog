# How to Use .env Files with Stacks in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, Environment Variables, DevOps

Description: Learn how to use .env files with Docker Compose stacks in Portainer for managing environment-specific configurations cleanly.

## Introduction

Docker Compose supports loading environment variables from a `.env` file automatically when it resides in the same directory as the `docker-compose.yml`. In Portainer's context, `.env` files can be used with Git-based stacks (where the `.env` file lives in the repository) or uploaded alongside a Compose file. Understanding how `.env` files interact with Portainer's environment variable system helps you choose the right approach for each environment.

## Prerequisites

- Portainer with an existing stack or ability to create one
- Understanding of `${VARIABLE}` substitution in Docker Compose

## How .env Files Work in Docker Compose

```
.env file (auto-loaded by Docker Compose):
  DB_PASSWORD=secret
  IMAGE_TAG=v1.2.3

docker-compose.yml references:
  image: myorg/app:${IMAGE_TAG}
  environment:
    - DB_PASSWORD=${DB_PASSWORD}

Result: Docker Compose substitutes variables before processing
```

## Step 1: Create a .env File

Structure your `.env` file with all stack variables:

```bash
# .env — loaded automatically by Docker Compose
# This file should NOT be committed to Git if it contains secrets

# Application settings
APP_NAME=myapp
ENVIRONMENT=production
IMAGE_TAG=v1.2.3

# Port configuration
WEB_PORT=80
API_PORT=8080
GRAFANA_PORT=3000

# Database configuration
DB_HOST=postgres
DB_PORT=5432
DB_NAME=myapp_db
DB_USER=appuser
DB_PASSWORD=change-this-password

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# Secrets (NEVER commit to git)
JWT_SECRET=change-this-jwt-secret
SMTP_PASSWORD=change-this-smtp-password
```

Always create a `.env.example` file for Git:

```bash
# .env.example — safe to commit, shows required variables
APP_NAME=myapp
ENVIRONMENT=production
IMAGE_TAG=latest

DB_HOST=postgres
DB_PORT=5432
DB_NAME=myapp_db
DB_USER=appuser
DB_PASSWORD=           # Set in deployment environment

JWT_SECRET=            # Set in deployment environment
SMTP_PASSWORD=         # Set in deployment environment
```

## Step 2: Use .env File with Git-Based Stacks

For Git-based stacks in Portainer:

**Option A: Commit a non-secret .env file to Git** (safe for non-sensitive configs):
```
repository/
├── docker-compose.yml
├── .env                  # Contains non-secret config, committed to Git
└── .env.example          # Template for secrets
```

Then in Portainer, set only the sensitive variables via the UI (they override `.env` file values).

**Option B: Do not commit .env, set everything in Portainer**:
```
repository/
├── docker-compose.yml
└── .env.example          # Template only — not the actual .env
```

Set all variables in Portainer's **Environment variables** section.

## Step 3: Portainer's env_file Directive

You can reference external env files within your Compose YAML:

```yaml
version: "3.8"

services:
  api:
    image: myorg/api:latest
    env_file:
      - ./config/api.env       # Loaded from the stack's directory
      - ./config/secrets.env   # Additional env file
    environment:
      # These override env_file values:
      - LOG_LEVEL=${LOG_LEVEL:-info}
```

Note: `env_file` paths are relative to the Compose file location. For Git-based stacks, these files must exist in the repository.

## Step 4: Load .env Variables in Portainer's Advanced Mode

To use an existing `.env` file with any stack type:

1. Navigate to **Stacks** → create or edit a stack.
2. Scroll to **Environment variables**.
3. Click **Advanced mode**.
4. Copy the contents of your `.env` file and paste them in.
5. Remove or blank out any comment lines (lines starting with `#`).
6. Click **Simple mode** to verify parsing.
7. Remove lines with empty values for secrets — add those as separate entries.

## Step 5: Multiple Environment Files

Use different `.env` files for different deployment targets:

```bash
# .env.production
ENVIRONMENT=production
IMAGE_TAG=v1.2.3
DB_NAME=myapp_prod
LOG_LEVEL=warn
REPLICAS=3

# .env.staging
ENVIRONMENT=staging
IMAGE_TAG=v1.3.0-rc1
DB_NAME=myapp_staging
LOG_LEVEL=debug
REPLICAS=1
```

When creating stacks in Portainer:
- For production stack: paste contents of `.env.production` in Advanced mode.
- For staging stack: paste contents of `.env.staging`.

## Step 6: Verify Variable Substitution

```bash
# Test locally before deploying to Portainer:
# Load variables and check what Compose generates:
docker compose --env-file .env.production config

# This shows the fully-substituted Compose file — confirms all variables resolve

# Check for missing variables (undefined with no default):
docker compose config 2>&1 | grep "variable is not set"
```

## Step 7: .gitignore for .env Files

Ensure secrets don't leak via Git:

```bash
# .gitignore entries:
.env
.env.local
.env.production
.env.staging
*.env.local

# Always commit .env.example
# NEVER commit actual .env with secrets
```

## Conclusion

`.env` files and Docker Compose variable substitution provide a clean pattern for managing configuration across environments. For Portainer stacks, the practical approach is to keep non-sensitive defaults in the `.env.example` (committed to Git), and inject secrets via Portainer's environment variable UI (never committed). Portainer's Advanced mode accepts the standard `KEY=VALUE` format, making it easy to load a full environment configuration at once. Always test variable substitution locally with `docker compose config` before deploying to confirm everything resolves correctly.
