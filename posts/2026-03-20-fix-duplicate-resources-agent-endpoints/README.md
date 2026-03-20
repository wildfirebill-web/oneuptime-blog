# How to Fix Duplicate Resources Appearing with Agent Endpoints

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Agent, Docker, Duplicate Containers, Endpoint

Description: Learn how to fix duplicate containers, volumes, or networks appearing in Portainer when using Agent endpoints, caused by multiple agent registrations or snapshot conflicts.

---

Duplicate resources in Portainer - seeing the same container listed twice - typically happen when the same Docker host is registered as multiple environments, or when the Portainer Agent is running multiple instances on the same host.

## Step 1: Identify Duplicate Environments

In Portainer go to **Environments**. Look for two environments pointing to the same host IP or with very similar names. Having both a "Local" socket connection and an Agent connection to the same host causes everything to appear twice.

## Step 2: Check for Multiple Agent Containers

```bash
# On the Docker host, check if multiple agent instances are running

docker ps | grep portainer_agent

# If you see more than one agent container, remove the extras
docker stop <extra-agent-container-id>
docker rm <extra-agent-container-id>
```

## Step 3: Remove the Duplicate Environment

In Portainer, remove the duplicate endpoint keeping only the one you want:

1. Go to **Environments**.
2. Identify the duplicate (check the URL/IP).
3. Click the environment settings and choose **Remove**.
4. Confirm removal.

## Step 4: Avoid Socket + Agent on the Same Host

A common mistake is adding the Portainer host as both a local socket endpoint and an agent endpoint. Remove one:

- Keep the local socket (`/var/run/docker.sock`) for the host running Portainer
- Use an Agent endpoint only for remote hosts

## Step 5: Clean Up Stale Environments

After a hardware migration or hostname change, old stale environments can interfere:

```bash
# In Portainer: Environments > select stale environment > Remove
# Then re-add with the correct new IP/hostname
```

## Step 6: Reset Snapshot Cache

If duplicates persist in the UI despite removing the extra environment, reset the snapshot by restarting Portainer:

```bash
docker restart portainer
```

Portainer will rebuild snapshots for all registered environments on restart, clearing stale cached data.
