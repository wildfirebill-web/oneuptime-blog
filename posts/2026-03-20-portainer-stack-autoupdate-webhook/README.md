# How to Set Up Stack Auto-Updates from Git in Portainer (Webhook)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, GitOps, Webhook, DevOps

Description: Learn how to configure Portainer stack webhooks triggered by Git pushes for instant redeployment without polling delays.

## Introduction

While polling auto-update checks for changes at regular intervals, webhook-based auto-update triggers an immediate redeployment the moment a push is made to your Git repository. Portainer generates a unique webhook URL for each stack; your Git provider calls this URL on push, and Portainer immediately pulls the latest Compose file and redeploys. This eliminates polling latency and reduces API calls to your Git provider.

## Prerequisites

- Portainer BE (Business Edition) or CE with webhook support
- A Git-based stack in Portainer
- Admin access to the Git repository (to configure webhooks)

## How Webhook Auto-Update Works

```
1. Developer pushes to Git repository
2. GitHub/GitLab/Bitbucket sends HTTP POST to Portainer webhook URL
3. Portainer receives the request, pulls latest Compose from Git
4. Portainer compares new content to deployed state
5. Changed services are recreated immediately
```

## Step 1: Enable Webhook on a Git-Based Stack

1. Navigate to **Stacks** → click the stack name.
2. Under **Automatic updates**, select **Webhook** (instead of Polling).
3. Portainer generates and displays the webhook URL:

```
https://your-portainer.example.com/api/stacks/webhooks/abc123def456...
```

4. Copy this URL.
5. Click **Save settings**.

## Step 2: Configure the Webhook in GitHub

1. Go to your repository on GitHub.
2. Navigate to **Settings** → **Webhooks** → **Add webhook**.
3. Configure:

```
Payload URL:   https://your-portainer.example.com/api/stacks/webhooks/abc123def456
Content type:  application/json
Secret:        (leave blank or set a secret)
Events:        Just the push event
Active:        ✓ checked
```

4. Click **Add webhook**.
5. GitHub sends a ping event — verify it shows a green checkmark.

## Step 3: Configure the Webhook in GitLab

1. Go to your project in GitLab.
2. Navigate to **Settings** → **Webhooks**.
3. Configure:

```
URL:           https://your-portainer.example.com/api/stacks/webhooks/abc123def456
Secret token:  (optional)
Trigger:       Push events
               Branch filter: main
SSL verify:    ✓ (if using valid SSL certificate)
```

4. Click **Add webhook**.
5. Click **Test** → **Push events** to verify connectivity.

## Step 4: Automate with GitHub Actions

Trigger the Portainer webhook as part of a CI/CD pipeline:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  deploy:
    name: Deploy Stack
    runs-on: ubuntu-latest
    steps:
      - name: Notify Portainer to redeploy
        run: |
          curl -X POST \
            -H "Content-Type: application/json" \
            "${{ secrets.PORTAINER_WEBHOOK_URL }}"
        # PORTAINER_WEBHOOK_URL is stored as a GitHub secret
```

This approach gives you more control: you can add test steps before the deployment trigger, and only call the webhook after tests pass.

## Step 5: Chain Webhook with Image Build

For a complete CI/CD pipeline:

```yaml
# .github/workflows/build-and-deploy.yml
name: Build, Push, and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myorg/api:${{ github.sha }}

      - name: Update image tag in Compose file
        run: |
          sed -i "s|image: myorg/api:.*|image: myorg/api:${{ github.sha }}|" docker-compose.yml
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add docker-compose.yml
          git commit -m "Update API image to ${{ github.sha }}"
          git push

      # Step above pushed to main — the webhook fires automatically
      # No need to call the webhook manually; GitHub fires it on push
```

## Step 6: Secure the Webhook

Protect the webhook endpoint from unauthorized access:

```bash
# 1. Webhook URL contains a secret token — treat it like a password
#    Never commit it to your repository

# 2. Store in GitHub as a repository secret:
#    Settings → Secrets → Actions → New repository secret
#    Name: PORTAINER_WEBHOOK_URL
#    Value: https://portainer.example.com/api/stacks/webhooks/abc123...

# 3. Restrict Portainer webhook to known IP ranges (firewall):
#    GitHub IP ranges: https://api.github.com/meta
#    GitLab IP ranges: https://docs.gitlab.com/ee/user/gitlab_com/

# 4. Use an nginx reverse proxy in front of Portainer with IP allowlisting
```

## Step 7: Test the Webhook Manually

```bash
# Trigger the webhook manually to verify it works:
curl -X POST \
  "https://your-portainer.example.com/api/stacks/webhooks/abc123def456"

# Expected response: HTTP 200 OK
# The stack will redeploy immediately
```

## Conclusion

Webhook-based auto-updates provide instant redeployment with zero polling overhead. Configure Portainer to generate a webhook URL, add it to your Git repository's webhook settings, and every push triggers an immediate redeploy. For the most complete CI/CD workflow, combine this with GitHub Actions or GitLab CI: build and push the image, update the image tag in the Compose file (or environment variables), push to Git, and let the webhook fire automatically. Store webhook URLs as CI/CD secrets — they are effectively authentication tokens.
