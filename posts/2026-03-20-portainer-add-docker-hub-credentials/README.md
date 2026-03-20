# How to Add Docker Hub Credentials to Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Hub, Registry, Authentication, DevOps

Description: Learn how to add Docker Hub credentials to Portainer to pull private images and avoid rate limiting on public pulls.

## Introduction

Adding Docker Hub credentials to Portainer serves two purposes: it enables pulling private Docker Hub repositories, and it significantly increases the rate limit for image pulls (authenticated users get 200 pulls per 6 hours vs 100 for anonymous users). This guide covers adding and managing Docker Hub credentials in Portainer.

## Prerequisites

- Portainer CE or BE installed
- A Docker Hub account (free or paid)
- Admin access to Portainer

## Why Add Docker Hub Credentials

1. **Private repositories** - Without credentials, Portainer cannot pull images from private Docker Hub repos
2. **Rate limits** - Docker Hub limits unauthenticated pulls; authentication significantly increases limits
3. **Docker Hub Pro/Team** - Paid accounts have much higher rate limits
4. **Security** - Using a dedicated access token is safer than using your account password

## Step 1: Create a Docker Hub Access Token

Using an access token is more secure than using your Docker Hub password:

1. Log in to [hub.docker.com](https://hub.docker.com)
2. Click your profile icon → **Account Settings**
3. Click **Security** in the left menu
4. Click **New Access Token**
5. Enter a description (e.g., "Portainer - Production")
6. Select permissions:
   - **Read-only** - For pulling images only
   - **Read/Write** - If Portainer also needs to push
7. Click **Generate**
8. **Copy the token immediately** - it won't be shown again

## Step 2: Add Docker Hub Registry in Portainer

1. Log in to Portainer as admin
2. Click **Registries** in the left sidebar
3. Click **+ Add registry**
4. Select **Docker Hub**

## Step 3: Fill in Docker Hub Credentials

```bash
Registry type:    Docker Hub
Username:         your-docker-hub-username
Access token:     dckr_pat_xxxxx...   (your personal access token)
```

Or use your Docker Hub password if you don't have an access token:

```text
Username:         your-docker-hub-username
Password:         your-docker-hub-password
```

5. Click **Add registry**

## Step 4: Verify the Registry Works

After adding:

1. Navigate to a Docker environment
2. Go to **Containers → Add container**
3. In the **Image** field, type a private image name
4. The Docker Hub registry should be selectable in the registry dropdown
5. Pull test: deploy a private image to verify authentication works

```bash
# CLI verification

docker login -u your-username -p your-access-token
docker pull your-username/private-repo:tag
```

## Step 5: Use Credentials When Deploying

### Containers

When creating a container or deploying a stack, Portainer automatically uses the configured credentials for Docker Hub images.

### Stacks with Private Images

```yaml
version: "3.8"

services:
  app:
    image: your-org/private-app:latest   # Portainer uses stored Docker Hub creds
```

No additional configuration needed - Portainer automatically uses the stored credentials.

## Step 6: Configure Per-Environment Registry Access

In Portainer BE, you can control which teams/environments have access to specific registries:

1. Go to **Registries**
2. Click on a registry
3. Under **Registry access** (BE feature), select which environments can use this registry

## Checking Docker Hub Rate Limits

Monitor your pull rate limit status:

```bash
# Check rate limit status for your account
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)

curl -s --head -H "Authorization: Bearer $TOKEN" \
  https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest | \
  grep -i "ratelimit"

# Output:
# ratelimit-limit: 200;w=21600   (200 pulls per 6 hours for authenticated)
# ratelimit-remaining: 195;w=21600
```

## Troubleshooting

### Authentication Failed

```bash
Error response from daemon: pull access denied for myorg/myimage,
repository does not exist or may require 'docker login'
```

**Fixes:**
- Verify username and password/token are correct
- Check that the token has `read` permission
- Ensure you are accessing the correct repository (private vs public)

### Rate Limit Exceeded

```text
Error response from daemon: toomanyrequests: Too Many Requests.
Rate limit exceeded.
```

**Fixes:**
- Add Docker Hub credentials if not already configured
- Upgrade to Docker Hub Pro for higher limits
- Cache images locally using a registry mirror

### Setting Up a Registry Mirror

```json
// /etc/docker/daemon.json - Docker Hub mirror
{
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

## Conclusion

Adding Docker Hub credentials to Portainer is quick and provides immediate benefits: access to private repositories and increased pull rate limits. Using a personal access token instead of your account password is the recommended approach for better security. If you're pulling images frequently, consider also setting up a local registry cache to reduce external pulls entirely.
