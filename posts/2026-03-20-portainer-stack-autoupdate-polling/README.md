# How to Set Up Stack Auto-Updates from Git in Portainer (Polling)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, GitOps, Auto-Update, DevOps

Description: Learn how to configure Portainer to automatically poll a Git repository and redeploy your stack when the Compose file changes.

## Introduction

Portainer's Git polling auto-update feature checks your repository at a defined interval and automatically redeploys the stack when the commit hash changes. This creates a lightweight GitOps workflow: push a change to Git, and Portainer picks it up within the polling interval without any manual intervention. This approach is simple to configure and works with any Git provider, requiring no webhooks or special repository settings.

## Prerequisites

- Portainer BE (Business Edition) or Portainer CE 2.14+
- A Git repository with a Docker Compose file
- The stack deployed from Git in Portainer

## How Polling Auto-Update Works

```text
1. Portainer checks the Git repo every N minutes
2. Compares current commit SHA to the deployed commit SHA
3. If different: pulls the new Compose file and redeploys
4. Services with changed definitions are recreated
5. Unchanged services continue running
```

## Step 1: Create a Git-Based Stack with Polling Enabled

1. Navigate to **Stacks** → **Add stack**.
2. Select **Repository** as the build method.
3. Configure Git settings:

```text
Repository URL:  https://github.com/myorg/my-infra
Repository ref:  refs/heads/main
Compose path:    docker-compose.yml
```

4. Under **Automatic updates**, enable the toggle.
5. Set **Polling interval**: `5m` (5 minutes), `15m`, `1h`, etc.
6. Optionally enable **Force re-pull images** to always pull the latest image even if the tag hasn't changed.
7. Click **Deploy the stack**.

## Step 2: Enable Polling on an Existing Git Stack

For an already-deployed Git stack:

1. Navigate to **Stacks** → click the stack name.
2. Scroll to the **Git repository** section.
3. Under **Automatic updates**, enable **Polling**.
4. Set the interval.
5. Click **Update the stack**.

## Step 3: Configure the Polling Interval

Choose an interval based on how quickly you need changes deployed:

```text
1m  - Near real-time (high API usage for large fleets)
5m  - Standard for most development workflows
15m - Balanced for stable services
1h  - Production services with infrequent changes
24h - Very stable configurations
```

Portainer polls the Git API for each stack separately. With many stacks and short intervals, consider the API rate limits of your Git provider:

```text
GitHub: 60 requests/hour (unauthenticated), 5000/hour (authenticated)
→ For 10 stacks with 5m polling: 120 requests/hour - use authentication
```

## Step 4: Trigger a Deployment by Pushing to Git

To test the polling update:

```bash
# Make a change to the Compose file:

cd my-infra/
# Edit docker-compose.yml - e.g., update IMAGE_TAG

# Commit and push:
git add docker-compose.yml
git commit -m "Update API image to v1.3.0"
git push origin main

# Portainer will detect the change within the polling interval
# and automatically redeploy the stack
```

## Step 5: Force Re-Pull Images with Polling

Enable force re-pull to handle mutable tags like `latest`:

```yaml
# Compose file uses mutable tag:
services:
  api:
    image: myorg/api:latest   # Tag doesn't change, but digest might
```

With **Force re-pull images** enabled:
- Portainer runs `docker pull` before each redeployment check.
- If the image digest changed, the service is recreated with the new image.
- Without this, `latest` would never trigger a redeployment if the tag name doesn't change.

For production, prefer immutable tags and update the tag in Git:

```yaml
# Better practice: update the tag in Git to trigger redeployment
services:
  api:
    image: myorg/api:${IMAGE_TAG:-latest}
```

Change `IMAGE_TAG` in the stack environment variables or in the Compose file.

## Step 6: Monitor Auto-Update Activity

Check what Portainer deployed and when:

1. In Portainer, navigate to **Stacks** → click the stack name.
2. The **Git repository** section shows the current deployed commit SHA.
3. Compare with your latest Git commit to confirm it's up to date.

```bash
# Get latest commit SHA on main branch:
git ls-remote https://github.com/myorg/my-infra refs/heads/main

# Compare with what Portainer shows in the UI
```

## Step 7: Handle Polling for Multiple Environments

Deploy the same repo to multiple environments with different branches:

```text
Stack: myapp-production
  Branch: refs/heads/main
  Polling: 15m
  Force re-pull: false

Stack: myapp-staging
  Branch: refs/heads/develop
  Polling: 5m
  Force re-pull: true  (always use latest images on staging)
```

Push to `develop` → staging updates within 5 minutes.
Merge PR to `main` → production updates within 15 minutes.

## Conclusion

Git polling auto-updates in Portainer provide a zero-configuration GitOps workflow - no webhooks to set up, no CI/CD pipeline changes required. Set a polling interval, push to Git, and Portainer handles the rest. The trade-off is latency: changes take up to the interval duration to deploy. For faster deployments, use the webhook-based auto-update method instead. Use polling for stable services and environments where the interval latency is acceptable, and enable Force re-pull when using mutable image tags to ensure the latest image is always deployed.
