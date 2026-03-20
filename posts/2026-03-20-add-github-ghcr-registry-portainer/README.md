# How to Add GitHub Container Registry (GHCR) to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitHub, GHCR, Container Registry, CI/CD

Description: Learn how to connect GitHub Container Registry (ghcr.io) to Portainer for pulling images built with GitHub Actions.

## Overview

GitHub Container Registry (GHCR) at `ghcr.io` allows you to publish container images alongside your GitHub repositories. It integrates with GitHub Actions for automated builds. Portainer can authenticate with GHCR using a Personal Access Token (PAT).

## Creating a GitHub Personal Access Token

1. Go to **GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Select scopes:
   - `read:packages` - to pull images
   - `write:packages` - if Portainer also needs to push
   - `delete:packages` - optional, for cleanup
4. Generate and copy the token.

## Adding GHCR to Portainer

1. Go to **Settings > Registries** and click **Add registry**.
2. Select **Custom registry** (or **GitHub** if available).
3. Enter:
   - **Registry URL**: `ghcr.io`
   - **Username**: Your GitHub username
   - **Password**: Your Personal Access Token
4. Click **Add registry**.

## Testing GHCR Authentication

```bash
# Log in to GHCR via Docker CLI

echo $GITHUB_TOKEN | docker login ghcr.io \
  -u your-github-username \
  --password-stdin

# Pull an image to confirm access
docker pull ghcr.io/your-org/your-image:latest
```

## Using GHCR in a Stack Compose File

```yaml
version: "3.8"

services:
  app:
    # Portainer uses the stored GHCR credentials to pull this image
    image: ghcr.io/your-org/your-app:v1.2.0
    deploy:
      replicas: 2
```

## Building and Pushing to GHCR with GitHub Actions

```yaml
# .github/workflows/build-push.yml
name: Build and Push to GHCR

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}/app:latest
```

## Making Packages Public

By default, packages pushed to GHCR are private. To make them public:

1. Go to the package on GitHub.
2. Click **Package settings**.
3. Scroll to **Danger Zone** and click **Change visibility**.

## Conclusion

GHCR combined with GitHub Actions provides a seamless build-push-deploy pipeline. Adding GHCR credentials to Portainer closes the loop by enabling automatic image pulls during stack deployments.
