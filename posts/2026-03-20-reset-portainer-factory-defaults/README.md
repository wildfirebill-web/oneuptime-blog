# How to Reset Portainer to Factory Defaults

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Administration, Reset, Factory Defaults, Recovery, Docker

Description: Learn how to completely reset Portainer to factory defaults by removing the data volume, while safely preserving your running containers and stacks.

---

A factory reset of Portainer wipes all users, settings, registries, and environment configurations. The underlying Docker containers and stacks are unaffected since they live in Docker, not in Portainer's database. This is useful after a corrupted database or when re-deploying from scratch.

## What Gets Deleted vs. What Is Preserved

| Deleted (Portainer Data) | Preserved (Docker) |
|---|---|
| Admin and user accounts | Running containers |
| Registered environments | Docker volumes |
| Stack configurations | Docker networks |
| Registry credentials | Docker images |
| Team and role settings | Compose labels on containers |
| OAuth/LDAP configuration | |

## Step 1: Back Up Before Resetting (Optional)

```bash
# Create a backup of the current Portainer database
docker run --rm -v portainer_data:/data alpine \
  tar czf - /data > portainer-backup-$(date +%Y%m%d-%H%M%S).tar.gz

# Store this off-host in case you need to recover
```

## Step 2: Stop and Remove Portainer

```bash
# Stop the Portainer container
docker stop portainer

# Remove the container (not the data volume yet)
docker rm portainer
```

## Step 3: Delete the Data Volume

```bash
# Remove the Portainer data volume - this is the factory reset
docker volume rm portainer_data

# Verify it is gone
docker volume ls | grep portainer_data
```

## Step 4: Redeploy Portainer

```bash
# Create a fresh Portainer deployment
docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 \
  -p 9443:9443 \
  -p 8000:8000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Complete Initial Setup

Open `http://<host>:9000` and create a new admin account. Portainer will ask you to set up your first environment.

## Step 6: Re-link Running Containers to Portainer

Your containers are still running. To manage them via Portainer:

1. Add the local Docker environment: **Environments > Add Environment > Docker (Local)**.
2. Portainer will immediately show all running containers.
3. Re-deploy or link stacks as needed.

## Partial Reset: Reset Admin Password Only

If you only need to reset the admin password (not a full factory reset):

```bash
docker stop portainer
docker run --rm -v portainer_data:/data portainer/helper-reset-password:latest
docker start portainer
```
