# How to Migrate Portainer from Version 1.x to 2.x

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, migration, upgrade, v1, v2

Description: A guide to migrating from Portainer version 1.x to the modern 2.x architecture, covering data migration, configuration changes, and breaking changes.

## Overview

Portainer 2.x introduced a completely new architecture compared to 1.x, including a new authentication system, agent-based connectivity, and significantly expanded features. Portainer 1.x reached end-of-life and is no longer maintained. This guide covers the migration path from 1.x to 2.x.

## Key Differences: Portainer 1.x vs 2.x

| Aspect | Portainer 1.x | Portainer 2.x |
|---|---|---|
| Architecture | Monolithic | Multi-tier (Server + Agents) |
| Multi-node Docker Swarm | Direct socket mount | Portainer Agent required |
| Kubernetes support | None | Full |
| Authentication | Local only | Local, LDAP, OAuth (BE) |
| RBAC | Basic | Advanced (BE) |
| Stacks from Git | No | Yes |
| Port (HTTP) | 9000 | 9000 (deprecated) |
| Port (HTTPS) | None | 9443 |
| Data directory | /data | /data |

## What Is Migrated

Portainer 2.x can read some 1.x data:
- Admin/user accounts (with password reset required)
- Endpoint/environment configurations (must be re-added)
- **NOT migrated**: Stacks, templates, custom configurations

## What Must Be Reconfigured

- All environments (Docker, Swarm, Kubernetes)
- Stack deployments (must be redeployed)
- User permissions
- Templates
- Registry configurations

## Migration Process

### Step 1: Document Your 1.x Configuration

Before starting, document everything in your Portainer 1.x:

```bash
# List all endpoints configured in Portainer 1.x
# Screenshot or export from Portainer 1.x UI:
# - Endpoints list
# - User list
# - Teams and access policies
# - Stack definitions (copy docker-compose content)
# - Template configurations
```

### Step 2: Export Stack Definitions

In Portainer 1.x, export all stack compose files:

```
Portainer 1.x → Stacks → Each stack → Editor → Copy compose content
```

Save each to a file:
```bash
mkdir -p stacks-backup
# Save each stack's compose content as stacks-backup/<stack-name>.yml
```

### Step 3: Deploy Portainer 2.x

```bash
# Keep Portainer 1.x running during initial setup of 2.x
# Deploy 2.x on different ports temporarily

docker volume create portainer2_data

docker run -d \
  -p 8001:8000 \
  -p 9444:9443 \
  --name portainer2 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer2_data:/data \
  portainer/portainer-ce:latest
```

### Step 4: Configure Portainer 2.x

```
1. Access https://server:9444
2. Create admin account (set a new password)
3. Add environments:
   - For Docker Standalone: Local Docker socket
   - For Docker Swarm: Deploy Portainer Agent first
   - For remote Docker: Add remote endpoint

# Deploy Portainer Agent for Swarm
docker service create \
  --name portainer_agent \
  --constraint 'node.platform.os == linux' \
  --mode global \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --mount type=bind,src=/var/lib/docker/volumes,dst=/var/lib/docker/volumes \
  --network portainer_agent_network \
  -p 9001:9001 \
  portainer/agent:latest
```

### Step 5: Redeploy Stacks

In Portainer 2.x, redeploy each stack:

```
Portainer 2.x → Stacks → Add Stack
→ Paste saved docker-compose content
→ Configure environment variables
→ Deploy
```

### Step 6: Validate Migration

- Verify all environments are connected
- Test deploying containers
- Check user access
- Validate stack deployments

### Step 7: Decommission Portainer 1.x

```bash
# Stop and remove Portainer 1.x
docker stop portainer && docker rm portainer
docker volume rm portainer_data   # 1.x data (no longer needed)

# Rename Portainer 2.x to standard ports
docker stop portainer2 && docker rm portainer2
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer2_data:/data \
  portainer/portainer-ce:latest
```

## Conclusion

Migrating from Portainer 1.x to 2.x is not a direct in-place upgrade but rather a fresh installation with manual reconfiguration of environments, users, and stacks. The good news is that Portainer 2.x is significantly more capable with Kubernetes support, better security, and agent-based multi-node management. Plan for a maintenance window of 1-2 hours to complete the migration and validate all environments are functioning correctly.
