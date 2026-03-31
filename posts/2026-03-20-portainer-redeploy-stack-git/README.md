# How to Redeploy a Stack from a Git Repository in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stack, Git, GitOps, DevOps

Description: Learn how to connect Portainer stacks to a Git repository and trigger redeployments automatically or manually.

## Introduction

Portainer supports Git-based stack deployments, enabling a GitOps workflow where your Docker Compose files live in version control. You can manually trigger redeployments or configure automatic updates when new commits are pushed. This guide covers setting up and redeploying Git-backed stacks.

## Prerequisites

- Portainer CE 2.x or BE
- A Git repository (GitHub, GitLab, Gitea, etc.) containing a Docker Compose file
- Portainer network access to your Git host

## Step 1: Create a Git-Backed Stack

1. Go to **Stacks > Add stack**
2. Enter a stack name
3. Select **Repository** as the build method
4. Fill in the repository details:

| Field | Example |
|-------|---------|
| Repository URL | `https://github.com/myorg/myapp` |
| Repository reference | `refs/heads/main` |
| Compose file path | `docker-compose.yml` or `deploy/compose.yml` |
| Username | Your Git username (if private repo) |
| Personal access token | Your PAT (if private repo) |

5. Click **Deploy the stack**

## Step 2: Understand the Redeployment Options

Portainer provides two redeployment modes:

### Manual Redeployment

Pull the latest commit and redeploy on demand from the Portainer UI.

### Automatic Redeployment (Polling)

Portainer periodically checks the repository for new commits and automatically redeploys when changes are detected.

## Step 3: Configure Automatic Updates

1. When creating or editing the stack, scroll to **Automatic updates**
2. Enable **Auto update**
3. Set the **Fetch interval** (e.g., every 5 minutes):

```text
Fetch interval: 5m    # Check every 5 minutes
```

4. Optionally enable **Force redeployment** to always redeploy even if the image digest hasn't changed

Portainer compares the current deployed commit SHA with the repository's HEAD. If they differ, redeployment is triggered.

## Step 4: Use Webhook-Based Redeployment

For faster, event-driven updates, use Portainer's stack webhook:

1. Go to your stack's detail page
2. Toggle **Enable webhook** to ON
3. Copy the generated webhook URL:

```text
https://portainer.example.com:9443/api/stacks/webhooks/abc123def456...
```

### Configure the Webhook in GitHub

1. Go to your repository → **Settings → Webhooks → Add webhook**
2. Paste the Portainer webhook URL
3. Set Content type to `application/json`
4. Choose **Just the push event**
5. Click **Add webhook**

Now every push to the repository triggers an automatic redeploy.

### Trigger Manually via curl

```bash
# Trigger a stack redeploy via webhook

curl -X POST \
  "https://portainer.example.com:9443/api/stacks/webhooks/abc123def456" \
  --insecure

# Expected response: 200 OK
```

## Step 5: Manually Redeploy via the UI

1. Navigate to **Stacks**
2. Click on your Git-backed stack
3. Click the **Pull and redeploy** button
4. Confirm in the dialog

Portainer will:
1. Pull the latest commit from the configured branch
2. Pull any updated Docker images
3. Recreate changed containers

## Step 6: Pin to a Specific Commit or Tag

To deploy a specific version instead of always tracking HEAD:

1. Edit the stack
2. Change **Repository reference** from `refs/heads/main` to:

```text
refs/tags/v1.2.3           # Pin to a tag
refs/heads/release/1.2     # Track a release branch
abc1234def5678...          # Pin to a specific commit SHA
```

## Step 7: Use Environment-Specific Compose Files

Structure your repository to support multiple environments:

```text
myapp/
├── docker-compose.yml           # Base configuration
├── docker-compose.prod.yml      # Production overrides
├── docker-compose.staging.yml   # Staging overrides
└── .env.example
```

In Portainer, set the **Compose file path** to `docker-compose.prod.yml` for the production stack and `docker-compose.staging.yml` for staging.

## Security Considerations

- Use **Personal Access Tokens** with minimal scope (read-only) instead of passwords
- For self-hosted Git (Gitea, GitLab), use SSH keys when possible
- Store sensitive values as Portainer environment variables, not in the repository

```yaml
services:
  app:
    image: myapp:latest
    environment:
      - SECRET_KEY=${SECRET_KEY}   # Set in Portainer, not in Git
```

## Conclusion

Git-backed stacks in Portainer bring GitOps principles to Docker-based deployments. Whether you prefer polling-based automatic updates or webhook-triggered redeployments, Portainer makes it easy to keep your stacks in sync with your source code repository. Combine this with environment-specific Compose files for a robust multi-environment deployment workflow.
