# How to Migrate Portainer Data Between Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Migration, Backup, Infrastructure

Description: Migrate Portainer configuration and data between servers with minimal downtime, including environment re-registration and agent reconnection procedures.

## Introduction

Migrating Portainer to a new server involves transferring the data volume (containing all configuration), updating environment URLs if the server address changes, and reconnecting agents to the new server. This guide provides a complete migration procedure.

## Migration Checklist

Before starting:
- [ ] New server has Docker installed
- [ ] New server has sufficient resources
- [ ] You have SSH access to both servers
- [ ] Backup of current Portainer data
- [ ] DNS or IP update plan ready

## Step 1: Create a Full Backup on Source Server

```bash
# On source server — stop Portainer for consistent backup
docker stop portainer

# Create backup
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine tar czf /tmp/portainer-migration.tar.gz -C /data .

# Start Portainer again (minimize downtime)
docker start portainer

echo "Backup created: /tmp/portainer-migration.tar.gz"
ls -lh /tmp/portainer-migration.tar.gz
```

## Step 2: Export Stack Definitions

```bash
# Export all stack compose files as a safety net
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

mkdir -p /tmp/portainer-stacks-export

for STACK_ID in $(curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/stacks | jq -r '.[].Id'); do

  STACK_NAME=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "http://localhost:9000/api/stacks/$STACK_ID" | jq -r '.Name')

  curl -s -H "Authorization: Bearer $TOKEN" \
    "http://localhost:9000/api/stacks/$STACK_ID/file" | \
    jq -r '.StackFileContent' > "/tmp/portainer-stacks-export/$STACK_NAME.yml"

  echo "Exported: $STACK_NAME"
done
```

## Step 3: Transfer Data to New Server

```bash
# Transfer backup and stack exports to new server
scp /tmp/portainer-migration.tar.gz user@new-server:/tmp/
scp -r /tmp/portainer-stacks-export/ user@new-server:/tmp/

# Verify transfer
ssh user@new-server "ls -lh /tmp/portainer-migration.tar.gz"
```

## Step 4: Restore on New Server

```bash
# On the NEW server:

# Create the data volume
docker volume create portainer_data

# Extract backup into the volume
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine tar xzf /backup/portainer-migration.tar.gz -C /data

# Verify restoration
docker run --rm \
  -v portainer_data:/data \
  alpine ls -la /data/

# Start Portainer
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  -p 8000:8000 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Test access
sleep 10
curl http://localhost:9000/api/status | jq .
```

## Step 5: Update DNS or IP

```bash
# If using a domain for Portainer:
# Update DNS A record to point to new server IP

# Verify DNS propagation
nslookup portainer.yourdomain.com
# or
dig portainer.yourdomain.com

# If using direct IP access, update bookmarks/links
```

## Step 6: Update Environment URLs

After migration, environment URLs may need updating if:
- The Portainer server has a new IP or hostname
- Agent endpoints reference the old server address

```bash
# Log in to the new Portainer instance
# Go to: Environments → Edit each environment
# Update the URL to the new server address if needed

# Via API
NEW_SERVER_IP="192.168.1.200"
TOKEN=$(curl -s -X POST http://$NEW_SERVER_IP:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# List all endpoints
curl -s -H "Authorization: Bearer $TOKEN" \
  http://$NEW_SERVER_IP:9000/api/endpoints | \
  jq '.[] | {id: .Id, name: .Name, url: .URL}'
```

## Step 7: Reconnect Agents

Remote agents store the Portainer server address. After migration, agents need to reconnect:

```bash
# Option A: Agents auto-reconnect if they can reach the new server
# (Works if the server kept the same IP/hostname)

# Option B: Update agent configuration if server address changed
# SSH to each agent host:
ssh agent-host

# Restart agent with new server URL
docker stop portainer-agent && docker rm portainer-agent
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# In Portainer (new server), the environment will show online
# once the agent starts and is reachable
```

## Step 8: Update Edge Agent Keys

Edge agents have the server URL encoded in their key. After migration:

```bash
# In new Portainer server, edge environments will show as offline
# You need to:
# 1. Go to Environments → select the edge environment
# 2. Get the new deployment command (which has the new server URL)
# 3. Redeploy the edge agent on each edge device

# Alternatively, if the server URL hasn't changed, just wait
# Edge agents will reconnect when they next check in
```

## Step 9: Decommission Old Server

```bash
# Once migration is verified:

# 1. Wait 24-48 hours to ensure everything works on new server

# 2. On old server, stop Portainer
docker stop portainer
docker rm portainer

# 3. Keep backup files on old server for 30 days
# then decommission

# 4. Update any DNS records pointing to old server

# 5. Notify users of the new server address if direct IP
```

## Step 10: Post-Migration Verification

```bash
# Comprehensive post-migration check
NEW_SERVER="http://new-server-ip:9000"

TOKEN=$(curl -s -X POST "$NEW_SERVER/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

echo "=== Post-Migration Check ==="

echo "Version:"
curl -s "$NEW_SERVER/api/status" | jq '.Version'

echo "Environments:"
curl -s -H "Authorization: Bearer $TOKEN" \
  "$NEW_SERVER/api/endpoints" | jq '.[].Name'

echo "Stacks:"
curl -s -H "Authorization: Bearer $TOKEN" \
  "$NEW_SERVER/api/stacks" | jq '.[].Name'

echo "Users:"
curl -s -H "Authorization: Bearer $TOKEN" \
  "$NEW_SERVER/api/users" | jq '.[].Username'

echo "Migration check complete"
```

## Conclusion

Portainer migration between servers follows a simple pattern: backup the data volume on the source, transfer to the destination, restore and start Portainer. The main post-migration tasks are updating environment URLs if the server address changed and reconnecting agents. Edge agents require special attention as the server URL is encoded in their deployment key and they may need to be redeployed with the new server address.
