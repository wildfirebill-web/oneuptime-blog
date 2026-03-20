# How to Migrate Containers Between Portainer Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Migration, Container Management, DevOps, Environments

Description: Learn how to move containers and stacks from one Portainer environment to another, including volume data and configuration transfer.

---

Portainer makes multi-environment management easy, but moving workloads between environments — for example, from staging to production, or from one Docker host to another — requires a methodical approach. This guide covers migrating containers, stacks, and their data between Portainer-managed environments.

---

## Understanding Portainer Environments

In Portainer, each Docker host or Kubernetes cluster is a separate "environment." Containers in one environment cannot be directly moved to another from the UI — you need to export configurations and transfer data manually.

---

## Step 1: Export the Stack Definition

The cleanest way to migrate is to export your stack's Compose file from the source environment.

In Portainer UI:
1. Go to the source environment
2. Navigate to **Stacks**
3. Click the stack you want to migrate
4. Click **Editor** to view the Compose YAML
5. Copy or download the Compose file

Alternatively, if you have access to the host filesystem:

```bash
# Find the Portainer-managed stack files on the source host
ls /var/lib/docker/volumes/portainer_data/_data/compose/

# Copy the specific stack directory
cp -r /var/lib/docker/volumes/portainer_data/_data/compose/5/ ~/stack-export/
```

---

## Step 2: Export Volume Data

For containers with persistent data, export each volume's contents.

```bash
# Export a named Docker volume to a tar archive
docker run --rm \
  -v my_app_data:/source \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/my_app_data.tar.gz -C /source .

# List what's in the archive to confirm
tar tzf my_app_data.tar.gz | head -20
```

---

## Step 3: Push Images to a Shared Registry

If custom images are used, ensure they're in a registry accessible from the target environment.

```bash
# Tag your image for a registry accessible to both environments
docker tag my-custom-app:v1 registry.example.com/my-custom-app:v1

# Push to the shared registry
docker push registry.example.com/my-custom-app:v1
```

---

## Step 4: Transfer Volume Data to the Target Host

```bash
# Transfer volume backup to the target server
scp my_app_data.tar.gz user@target-host:/home/user/

# On the target host: create the volume and restore
docker volume create my_app_data

docker run --rm \
  -v my_app_data:/target \
  -v /home/user:/backup \
  alpine \
  tar xzf /backup/my_app_data.tar.gz -C /target
```

---

## Step 5: Deploy the Stack in the Target Environment

In the Portainer UI for the target environment:

1. Go to **Stacks > Add Stack**
2. Give it the same name as the source stack
3. Paste the Compose YAML you exported
4. Update any environment variables specific to the target (database URLs, hostnames, etc.)
5. Click **Deploy the stack**

```yaml
# Updated compose for target environment
# Key changes: image registry URL updated, env vars for new environment
version: "3.8"

services:
  app:
    image: registry.example.com/my-custom-app:v1  # updated registry URL
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db-production.internal  # updated for target environment
      - APP_ENV=production
    volumes:
      - my_app_data:/app/data

  db:
    image: postgres:15
    restart: unless-stopped
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  my_app_data:
    external: true  # use the volume we pre-populated
  db_data:
```

---

## Step 6: Validate the Migration

```bash
# On the target host, verify containers are running
docker ps

# Check container logs for errors
docker logs -f <container_name>

# Verify volume data is accessible
docker exec <container_name> ls /app/data
```

---

## Summary

Migrating containers between Portainer environments is a four-step process: export the Compose file, back up volume data, transfer data to the target host, and redeploy the stack. Update environment-specific settings in the Compose file before deploying to the target environment to avoid configuration drift.
