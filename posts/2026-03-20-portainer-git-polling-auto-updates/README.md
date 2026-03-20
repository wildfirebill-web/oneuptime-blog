# How to Configure Git Polling for Auto-Updates in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitOps, Docker, Automation, Polling

Description: Learn how to configure Git polling in Portainer to automatically detect and deploy changes from a Git repository at regular intervals without manual intervention.

## Introduction

Git polling is Portainer's mechanism for periodically checking your Git repository for changes and automatically redeploying your stack when it detects new commits. Unlike webhook-based updates (which require your Git host to reach Portainer), polling works in any network environment, including setups where Portainer is behind a firewall.

## Prerequisites

- Portainer CE or BE
- A stack deployed from a Git repository
- The Git repository must be accessible from the Portainer server

## How Git Polling Works

1. Portainer records the commit hash at the time of deployment
2. At each polling interval, it fetches the latest commit hash from the configured branch
3. If the hash differs, it triggers a pull and redeploy of the stack
4. The process is entirely pull-based - no inbound connections required

## Step 1: Enable Polling on an Existing Stack

1. Log into Portainer.
2. Go to **Stacks** and click on your Git-connected stack.
3. Click **Edit stack** or the **GitOps updates** section.
4. Enable the **Fetch updates** toggle.
5. Set the polling interval.
6. Click **Save**.

## Step 2: Choose the Right Polling Interval

| Interval | Use Case |
|----------|----------|
| `1m` | Active development - fast feedback, high server load |
| `5m` | Standard CI/CD - good balance of speed and load |
| `15m` | Stable applications - moderate update frequency |
| `1h` | Configuration drift prevention - for rarely-changed stacks |
| `24h` | Daily sync - for very stable production environments |

## Step 3: Configure Polling via the Portainer API

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-auth-token"
ENDPOINT_ID=1
STACK_ID=3

# Enable polling with a 5-minute interval

curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}?endpointId=${ENDPOINT_ID}" \
  -d '{
    "AutoUpdate": {
      "FetchInterval": "5m",
      "Webhook": "",
      "ForceUpdate": false
    },
    "pullImage": true,
    "prune": false
  }' | jq .
```

## Step 4: Force Redeployment on Every Poll

By default, Portainer only redeploys when the Git commit hash changes. Enable `ForceUpdate` to always redeploy even without new commits (useful for pulling latest image tags):

```bash
# Enable forced redeployment on every poll
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}?endpointId=${ENDPOINT_ID}" \
  -d '{
    "AutoUpdate": {
      "FetchInterval": "5m",
      "ForceUpdate": true
    },
    "pullImage": true
  }' | jq .
```

> **When to use ForceUpdate**: If your Compose file uses `:latest` tags and you want to always pull fresh images, enable this. If you use pinned image versions and only redeploy on actual config changes, keep it disabled.

## Step 5: Deploy a Stack with Polling Configured

When creating a new Git-connected stack via the API:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/create/standalone/repository?endpointId=${ENDPOINT_ID}" \
  -d '{
    "name": "my-app",
    "repositoryURL": "https://github.com/your-org/my-app",
    "repositoryReferenceName": "refs/heads/main",
    "filePathInRepository": "docker-compose.yml",
    "repositoryAuthentication": false,
    "autoUpdate": {
      "interval": "5m",
      "forcePullImage": true,
      "forceUpdate": false
    },
    "env": []
  }' | jq .
```

## Step 6: Monitor Polling Activity

Check when the last poll occurred and its result:

```bash
# Get stack details including last update info
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}" | \
  jq '{
    name: .Name,
    git_hash: .GitConfig.ConfigHash,
    last_poll: .GitConfig.LastPollTime,
    auto_update: .AutoUpdate
  }'
```

Check Portainer logs for polling activity:

```bash
# View Portainer container logs
docker logs portainer 2>&1 | grep -i "polling\|git\|update" | tail -20
```

## Step 7: Disable Polling

To stop automatic updates:

1. In the Portainer UI, go to the stack's GitOps settings.
2. Disable the **Fetch updates** toggle.
3. Save.

Via API:

```bash
# Disable auto-update by setting interval to empty
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}?endpointId=${ENDPOINT_ID}" \
  -d '{
    "AutoUpdate": {
      "FetchInterval": "",
      "ForceUpdate": false
    }
  }' | jq .
```

## Step 8: Polling vs Webhook Comparison

| Feature | Polling | Webhook |
|---------|---------|---------|
| Network requirement | Portainer → Git (outbound) | Git → Portainer (inbound) |
| Firewall-friendly | Yes | Requires open port/reverse proxy |
| Update latency | 1-60 minutes | Near-instant (seconds) |
| Git host support | Any | Requires webhook capability |
| Configuration | Simple (interval setting) | Requires webhook setup in Git |

Use **polling** when:
- Portainer is behind a firewall or NAT
- You want simpler setup
- Update latency is acceptable

Use **webhooks** when:
- Instant deployment on push is required
- Portainer is publicly accessible
- You want to minimize unnecessary polling requests

## Conclusion

Git polling in Portainer provides a reliable, pull-based mechanism for automatic stack updates. Configure an appropriate polling interval based on your update frequency requirements, enable force redeployment if using mutable image tags, and monitor polling activity through Portainer logs. For environments where inbound webhooks are possible, consider adding webhooks alongside polling for faster initial response with polling as a backup.
