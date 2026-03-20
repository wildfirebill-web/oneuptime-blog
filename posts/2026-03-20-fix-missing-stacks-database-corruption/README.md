# How to Fix Missing Stacks After Portainer Database Corruption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Database Corruption, BoltDB, Stacks, Recovery

Description: Learn how to recover missing stacks after Portainer's BoltDB database becomes corrupted, including inspection, partial recovery, and prevention strategies.

---

Portainer uses BoltDB for persistent storage. Power failures, OOM kills, or disk errors can corrupt the database file, causing all stacks, users, and settings to disappear after restart. This guide covers recovery strategies.

## Detecting Database Corruption

```bash
# Check Portainer logs for BoltDB errors
docker logs portainer 2>&1 | grep -i "bolt\|corrupt\|database\|unexpected"

# Common corruption messages:
# "unexpected end of file"
# "bolt: database file might be corrupted"
# "panic: db file size has grown without permission"
```

## Step 1: Attempt BoltDB Consistency Check

```bash
# Install bolt CLI tool
go install go.etcd.io/bbolt/cmd/bbolt@latest

# Or use the Docker image
docker run --rm -v portainer_data:/data alpine/bbolt \
  check /data/portainer.db

# Output will indicate if the file is consistent or not
```

## Step 2: Try to Salvage Data from Corrupted DB

```bash
# Attempt to export what is still readable from the database
docker run --rm -v portainer_data:/data alpine/bbolt \
  pages /data/portainer.db 2>/dev/null | head -50

# If the file is partially readable, try dumping specific buckets
# The stack bucket in Portainer is named "stacks"
```

## Step 3: Recover from a Backup

The cleanest recovery path is from a backup:

```bash
# Stop Portainer
docker stop portainer

# Restore the backup into the volume
docker run --rm \
  -v portainer_data:/data \
  -v /path/to/backup:/backup \
  alpine \
  tar xzf /backup/portainer-backup.tar.gz -C /

# Start Portainer
docker start portainer
```

## Step 4: Rebuild by Re-deploying Stacks from Git

If stacks were deployed from Git repositories, recovery is straightforward:

1. Start Portainer fresh (new empty database).
2. Create an admin user.
3. Re-add environments.
4. Re-deploy each stack from its Git repository.

The containers and volumes are still running — only the Portainer metadata is lost.

## Step 5: Extract Compose from Running Containers

For stacks not stored in Git, reconstruct the compose from running container labels:

```bash
# List all containers with their compose project labels
docker ps -a --format '{{.Names}}' | while read name; do
  project=$(docker inspect "$name" --format '{{index .Config.Labels "com.docker.compose.project"}}' 2>/dev/null)
  [ -n "$project" ] && echo "Container: $name, Project: $project"
done
```

## Prevention Strategy

Schedule automated backups before this ever happens again:

```bash
# Add to crontab: daily backup at 2 AM
0 2 * * * docker run --rm -v portainer_data:/data alpine tar czf - /data \
  > /mnt/backups/portainer-$(date +\%Y\%m\%d).tar.gz
```
