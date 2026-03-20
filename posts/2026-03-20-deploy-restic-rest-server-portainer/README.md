# How to Deploy Restic with REST Server via Portainer - Rest Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Restic, Backup, Docker, Storage, Security

Description: Deploy the Restic REST Server as a Docker container via Portainer to host a local backup repository, then configure Restic clients to push encrypted backups to it.

---

Restic is a fast, secure backup tool that deduplicates and encrypts data before storing it. The Restic REST Server provides an HTTP backend for remote repositories, making it easy to centralize backups from multiple hosts. Deploying the REST server via Portainer gives you a managed backup target with persistent storage.

## Architecture

```mermaid
graph LR
    Client1[App Server 1] -->|HTTPS| REST[Restic REST Server]
    Client2[App Server 2] -->|HTTPS| REST
    REST --> Repo[(Repository Volume)]
```

## Step 1: Deploy Restic REST Server via Portainer

Go to **Stacks > Add Stack** in Portainer:

```yaml
# restic-rest-server-stack.yml

version: "3.8"

services:
  restic-rest:
    image: restic/rest-server:latest
    environment:
      # Enable authentication - create htpasswd users separately
      - OPTIONS=--htpasswd-file /data/.htpasswd --tls
    ports:
      - "8000:8000"   # Restic REST API endpoint
    volumes:
      # Repository data storage
      - restic-data:/data
    restart: unless-stopped

volumes:
  restic-data:
```

## Step 2: Create Repository Users

Access the container via Portainer's **Console** tab and generate an htpasswd entry:

```bash
# Inside the container - create a backup user
htpasswd -B -c /data/.htpasswd backupuser
```

## Step 3: Initialize the Repository from a Client

From any host running the Restic CLI:

```bash
# Initialize a new repository against the REST server
export RESTIC_REPOSITORY=rest:https://backupuser:password@<server-ip>:8000/myrepo
export RESTIC_PASSWORD=encryption-passphrase

restic init
```

## Step 4: Run Backups

```bash
# Back up /opt/appdata to the remote repository
restic backup /opt/appdata

# List available snapshots
restic snapshots

# Restore a specific snapshot
restic restore latest --target /tmp/restore
```

## Step 5: Automate with Portainer Cron Jobs

Create a wrapper script on each client host and schedule it via a Portainer stack using a cron-style container:

```yaml
services:
  restic-backup:
    image: restic/restic:latest
    environment:
      - RESTIC_REPOSITORY=rest:https://backupuser:password@restic-rest:8000/myrepo
      - RESTIC_PASSWORD=encryption-passphrase
    command: >
      sh -c "restic backup /data &&
             restic forget --keep-daily 7 --keep-weekly 4 --prune"
    volumes:
      - /opt/appdata:/data:ro
    # Restart policy = no for one-shot containers
    restart: "no"
```

## Managing Retention

Restic's `forget` and `prune` commands implement retention policies:

```bash
# Keep 7 daily, 4 weekly, 6 monthly snapshots; remove the rest
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
```

## Summary

The Restic REST Server on Portainer gives you a self-hosted, TLS-secured backup repository. Clients encrypt data before transmission, so the server never sees plaintext. Combined with Restic's deduplication, storage usage stays minimal even with frequent backups.
