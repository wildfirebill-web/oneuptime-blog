# How to Fix Missing Stacks After Portainer Database Corruption - Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Database, Recovery, Stack

Description: Recover missing stacks and configuration after Portainer's BoltDB database becomes corrupted, including database repair and data recovery techniques.

## Introduction

Portainer uses BoltDB (an embedded key-value database) to store all configuration, including stack definitions, user accounts, environment settings, and access control rules. If this database becomes corrupted - typically due to an unclean shutdown, full disk, or storage failure - stacks and other configuration disappear. This guide explains recovery options.

## Step 1: Identify Database Corruption

```bash
# Check Portainer logs for BoltDB errors

docker logs portainer 2>&1 | grep -iE "corrupt|bolt|database|invalid|panic" | head -20

# Common corruption indicators:
# "invalid database"
# "panic: runtime error: invalid memory address"
# "bolt: timeout"
# "unexpected page type"
# "page 0: can't read page: unexpected EOF"
```

## Step 2: Check the Database File

```bash
# Check if the file exists and its size
docker run --rm \
  -v portainer_data:/data \
  alpine stat /data/portainer.db

# A zero-byte or very small portainer.db indicates corruption or truncation
# Normal size: a few MB (varies with number of stacks/users/etc)

# Check file integrity
docker run --rm \
  -v portainer_data:/data \
  alpine sh -c "ls -lh /data/ && md5sum /data/portainer.db"
```

## Step 3: Stop Portainer Before Attempting Recovery

```bash
# IMPORTANT: Stop Portainer before any database operations
# Writing to a corrupt database can make it worse
docker stop portainer
```

## Step 4: Backup the Corrupt Database

```bash
# Always backup before attempting repairs
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine cp /data/portainer.db /backup/portainer.db.corrupt.$(date +%Y%m%d%H%M%S)

echo "Backup saved to /tmp/portainer.db.corrupt.*"
```

## Step 5: Attempt BoltDB Repair

BoltDB has built-in consistency checking. Use the `bbolt` tool:

```bash
# Install bbolt (bbolt is the successor to bolt)
# Or use the bolt tool included in some Go distributions

# Option A: Use a pre-built binary
docker run --rm \
  -v portainer_data:/data \
  alpine sh -c "
    apk add --no-cache go &&
    go install go.etcd.io/bbolt/cmd/bbolt@latest &&
    /root/go/bin/bbolt check /data/portainer.db
  "

# Option B: Use bolt CLI tools
# Download from: https://github.com/etcd-io/bbolt/releases
# Run: bbolt check portainer.db
# If check passes, the database is intact
# If it reports errors, try: bbolt compact portainer.db repaired.db
```

## Step 6: Restore from Backup

The best recovery is from a backup:

```bash
# Stop Portainer
docker stop portainer

# Remove the corrupt database
docker run --rm \
  -v portainer_data:/data \
  alpine rm /data/portainer.db

# Restore from backup
docker run --rm \
  -v portainer_data:/data \
  -v /opt/backups/portainer:/backup \
  alpine sh -c "
    # Find the most recent backup
    LATEST=$(ls -t /backup/*.tar.gz | head -1)
    echo 'Restoring from: '$LATEST
    tar xzf \$LATEST -C /data
  "

# Start Portainer
docker start portainer
```

## Step 7: Recreate Stacks from Running Containers

If no backup exists but containers are still running:

```bash
# Find all running containers with Compose labels
docker ps --format "json" | jq -r '.Labels' 2>/dev/null

# List unique stack (project) names
docker ps -q | xargs docker inspect \
  --format '{{index .Config.Labels "com.docker.compose.project"}}' \
  | sort -u | grep -v "^$"

# For each stack, collect service names
docker ps -q | xargs docker inspect \
  --format '{{index .Config.Labels "com.docker.compose.project"}}: {{index .Config.Labels "com.docker.compose.service"}}' \
  | grep -v ": $" | sort
```

## Step 8: Reset Portainer and Re-Add Everything

If recovery is not possible:

```bash
# Delete the corrupt database
docker run --rm \
  -v portainer_data:/data \
  alpine rm /data/portainer.db

# Start Portainer - it will initialize fresh
docker start portainer

# Portainer is now fresh - re-add:
# 1. Environments (Docker hosts, Kubernetes clusters)
# 2. Users and teams
# 3. Registries
# 4. Stacks (re-create from compose files if you have them)
```

## Step 9: Recover Stack Definitions

If you stored compose files in Git (recommended):

```bash
# Stack definitions were version-controlled
cd /opt/stacks-git
git log --oneline | head -20
git checkout HEAD -- .

# Re-deploy stacks via Portainer UI or CLI
docker compose -f mystack.yml up -d
```

If not in Git, use docker-inspect to reconstruct:

```bash
# For each container in the stack
for CONTAINER in $(docker ps --format "{{.Names}}" | grep "stackname"); do
  echo "=== $CONTAINER ==="
  docker inspect $CONTAINER | jq '.[0] | {
    Image: .Config.Image,
    Env: .Config.Env,
    Ports: .HostConfig.PortBindings,
    Volumes: .HostConfig.Binds,
    Networks: [.NetworkSettings.Networks | keys[]]
  }'
done
```

## Step 10: Prevent Future Corruption

```bash
# 1. Enable live-restore in Docker to prevent unclean shutdowns
cat > /etc/docker/daemon.json << 'EOF'
{
  "live-restore": true
}
EOF

# 2. Use UPS or graceful shutdown scripts
# Add to systemd service override
sudo systemctl edit docker
# Add:
# [Service]
# ExecStop=/bin/bash -c 'docker stop portainer && sleep 2'

# 3. Set up automated backups (see backup posts in this series)

# 4. Use a volume on reliable storage (SSD, not NFS with poor write ordering)
```

## Conclusion

Portainer database corruption is serious but recoverable if you have backups. Without backups, the containers continue running independently of Portainer's database, so your workloads are safe - only the Portainer management metadata is lost. Prevent future corruption by setting up automated backups of the `portainer_data` volume and using `live-restore` in Docker to enable graceful handling of Docker daemon restarts.
