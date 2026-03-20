# How to Fix "Stack Not Found" After a Portainer Crash

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Stacks, Recovery, Database, Crash

Description: Learn how to recover from "Stack Not Found" errors after a Portainer crash by restoring stack metadata from the database or re-importing running stacks.

---

After a Portainer crash or improper shutdown, stack metadata in the BoltDB database can become inconsistent. Portainer may show "Stack Not Found" even though the containers are still running. This guide covers recovery options.

## Understanding the Problem

Portainer stores stack metadata (name, compose content, environment variables) in its BoltDB database (`portainer.db`). The actual running containers live in Docker — independent of Portainer. A corrupt database loses the metadata link but not the running containers.

## Step 1: Verify Containers Are Still Running

```bash
# Check if your stack containers are still running despite the error
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"

# List Docker-managed networks (stacks usually create a dedicated network)
docker network ls | grep <stack-name>
```

## Step 2: Check Portainer Logs

```bash
docker logs portainer --tail 50 | grep -i "stack\|error\|database"
```

## Step 3: Restore from Backup

If you have a Portainer backup:

1. In Portainer go to **Settings > Backup Portainer**.
2. Upload the `.tar.gz` backup file.
3. Click **Restore**.

Portainer will restore all stack metadata.

## Step 4: Re-import Orphaned Stacks

For stacks deployed from Portainer without a Git repository, you can re-import them:

```bash
# Find the Compose labels on a container from the stack
docker inspect <container-name> | python3 -c "
import json, sys
data = json.load(sys.stdin)
labels = data[0]['Config']['Labels']
for k, v in labels.items():
    if 'compose' in k.lower() or 'stack' in k.lower():
        print(f'{k}: {v}')"
```

Then in Portainer create a new stack with the same name and paste the compose content. Portainer will detect the existing containers and link them.

## Step 5: Use the --rollback Flag for Failed Upgrades

If the crash occurred during a Portainer upgrade:

```bash
# Roll back to the previous version
docker stop portainer && docker rm portainer

# Run the previous Portainer version (replace with your previous version)
docker run -d \
  --name portainer \
  portainer/portainer-ce:2.19.5 \
  --rollback-from 2.20.0
```

## Prevention: Back Up Before Updates

```bash
# Create a backup before any Portainer update
docker run --rm -v portainer_data:/data alpine \
  tar czf - /data > portainer-backup-$(date +%Y%m%d).tar.gz
```
