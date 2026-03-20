# How to Recover Portainer After a Failed Upgrade

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Upgrade, Recovery, Troubleshooting, Rollback, Administration

Description: Learn how to recover Portainer after a failed upgrade by rolling back to the previous version or restoring from a pre-upgrade backup.

---

A failed Portainer upgrade can leave the UI inaccessible or the database in a partially migrated state. This guide provides step-by-step recovery procedures.

## Signs of a Failed Upgrade

```bash
# Check Portainer logs for migration failures
docker logs portainer 2>&1 | grep -i "migration\|panic\|fatal\|error"

# Common failure messages:
# "FATAL: failed to migrate database"
# "panic: database migration error"
# "bolt: timeout"
```

## Option 1: Roll Back to Previous Image Version

If you have a backup:

```bash
# 1. Stop the broken Portainer
docker stop portainer && docker rm portainer

# 2. Note: Portainer may have already run partial migrations
# If the database is partially migrated, roll back the data volume too

# 3. Restore pre-upgrade backup (if you took one)
docker volume rm portainer_data
docker volume create portainer_data
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine tar xzpf /backup/portainer-pre-upgrade.tar.gz -C /

# 4. Start the previous version
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:2.19.5   # Use your previous working version
```

## Option 2: Force Database Reset

If you have no backup but the running containers are fine:

```bash
# Stop Portainer
docker stop portainer && docker rm portainer

# Remove only the database file (keeps agent keys and certs)
docker run --rm -v portainer_data:/data alpine rm /data/portainer.db

# Start the NEW version - it will create a fresh database
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# You will need to re-configure users, environments, and registries
```

## Option 3: Retry the Migration with Debug Logging

Sometimes a transient error during migration can be fixed by retrying:

```bash
docker stop portainer && docker rm portainer
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --log-level DEBUG

# Watch for migration success or the specific error
docker logs -f portainer 2>&1 | grep -i "migrat"
```

## Prevention: Always Backup Before Upgrading

```bash
# Pre-upgrade backup script
docker run --rm -v portainer_data:/data alpine tar czpf - /data > \
  portainer-pre-upgrade-$(date +%Y%m%d).tar.gz
echo "Backup ready. Proceeding with upgrade..."
```
