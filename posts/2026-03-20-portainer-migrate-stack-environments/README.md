# How to Migrate a Stack Between Environments in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, Migration, DevOps

Description: Learn how to migrate Docker Compose stacks from one environment to another in Portainer, including volume data migration and configuration transfer.

## Introduction

Moving a stack between environments — from development to staging, staging to production, or from one Docker host to another — requires migrating both the stack configuration and any persistent data in volumes. Portainer doesn't have a built-in "move stack" feature, but the process is straightforward: export the configuration, transfer the data, and redeploy in the target environment. This guide covers the complete migration workflow.

## Prerequisites

- Portainer managing both source and target environments
- SSH access or shared storage between hosts
- Docker CLI access on both hosts

## Migration Overview

```
Source Environment          Target Environment
────────────────────        ────────────────────
1. Export stack config  →   3. Import config
2. Backup volumes       →   4. Restore volumes
                            5. Deploy stack
                            6. Verify & test
```

## Step 1: Export the Stack Configuration

In Portainer (source environment):

1. Navigate to **Stacks** → click the stack name.
2. In the editor, copy the complete Compose YAML.
3. Note the environment variables (check the Variables section).
4. Save locally as `myapp-stack.yml`.

Or via CLI on the source host:

```bash
# Export using docker compose config (shows resolved variables):
cd /opt/stacks/myapp/
docker compose config > myapp-stack-resolved.yml

# The raw file (with variable placeholders):
cp docker-compose.yml myapp-stack.yml
```

## Step 2: Backup Volumes from the Source

```bash
# List volumes used by the stack:
docker compose ps -q | xargs docker inspect \
  --format '{{range .Mounts}}{{.Name}} {{end}}' | tr ' ' '\n' | sort -u

# Backup each volume as a tar archive:
docker run --rm \
  -v myapp_postgres_data:/source:ro \
  -v /backup:/backup \
  alpine tar czf /backup/postgres_data_$(date +%Y%m%d).tar.gz -C /source .

docker run --rm \
  -v myapp_app_uploads:/source:ro \
  -v /backup:/backup \
  alpine tar czf /backup/app_uploads_$(date +%Y%m%d).tar.gz -C /source .

# Verify backups were created:
ls -lh /backup/
```

## Step 3: Transfer Files to Target Host

```bash
# Transfer the stack config and volume backups:
scp myapp-stack.yml user@target-host:/opt/stacks/myapp/docker-compose.yml
scp /backup/postgres_data_*.tar.gz user@target-host:/backup/
scp /backup/app_uploads_*.tar.gz user@target-host:/backup/

# Or use rsync for large datasets:
rsync -avz --progress /backup/ user@target-host:/backup/
```

## Step 4: Restore Volumes on the Target Host

On the target host, before deploying the stack:

```bash
# Create volumes (Docker creates them on stack deploy, but we need them first):
docker volume create myapp_postgres_data
docker volume create myapp_app_uploads

# Restore volume data:
docker run --rm \
  -v myapp_postgres_data:/target \
  -v /backup:/backup:ro \
  alpine tar xzf /backup/postgres_data_20240101.tar.gz -C /target

docker run --rm \
  -v myapp_app_uploads:/target \
  -v /backup:/backup:ro \
  alpine tar xzf /backup/app_uploads_20240101.tar.gz -C /target

# Verify contents:
docker run --rm -v myapp_postgres_data:/vol alpine ls -la /vol
```

## Step 5: Deploy the Stack in Portainer (Target)

1. In Portainer, select the target environment.
2. Navigate to **Stacks** → **Add stack**.
3. Paste the exported Compose YAML.
4. Configure environment variables for the target environment:

```
# Source (staging) values:
DB_PASSWORD=staging_password
API_URL=https://staging-api.example.com

# Target (production) values:
DB_PASSWORD=production_password
API_URL=https://api.example.com
```

5. Click **Deploy the stack**.

## Step 6: Verify the Migration

```bash
# Check all containers are running:
docker ps --filter "label=com.docker.compose.project=myapp"

# Test the application:
curl https://api.example.com/health

# Check the database contains migrated data:
docker exec myapp_postgres_1 psql -U appuser -d myapp -c "SELECT count(*) FROM users;"

# Compare row counts with source:
# (run same query on source and compare)
```

## Step 7: Environment-Specific Configuration Changes

Update configurations that differ between environments:

```yaml
# docker-compose.yml — use variables for environment-specific values
services:
  nginx:
    image: nginx:alpine
    environment:
      - DOMAIN=${DOMAIN}           # staging.example.com vs example.com

  api:
    image: myorg/api:${IMAGE_TAG}
    environment:
      - LOG_LEVEL=${LOG_LEVEL}     # debug on staging, warn on production
      - SENTRY_DSN=${SENTRY_DSN}   # different DSN per environment
      - CORS_ORIGIN=${CORS_ORIGIN} # different origins per environment
```

Set correct values in Portainer's environment variables for each stack.

## Conclusion

Migrating a stack between Portainer environments requires three components: the Compose YAML configuration, environment-specific variable values, and volume data backups. Export the Compose file from the source, backup volumes using a temporary Alpine container, transfer both to the target host, restore volumes before deploying, and then create the stack in Portainer with target-appropriate environment variables. For stacks deployed from Git, only the volume data and environment variable changes are needed — the Compose file is already versioned in the repository.
