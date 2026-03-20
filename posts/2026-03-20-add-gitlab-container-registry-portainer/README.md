# How to Add GitLab Container Registry to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitLab, Container Registry, CI/CD, DevOps

Description: Learn how to connect GitLab's built-in container registry to Portainer for pulling CI-built images.

## Overview

GitLab includes a built-in container registry that integrates with GitLab CI/CD pipelines. Every GitLab project can push images to `registry.gitlab.com/<namespace>/<project>`. Portainer can pull from this registry using a GitLab personal access token or deploy token.

## Creating a GitLab Deploy Token

Deploy tokens are the preferred way to grant Portainer read access to your GitLab registry:

1. In GitLab, go to your project's **Settings > Repository**.
2. Expand **Deploy tokens**.
3. Create a new token with:
   - **Name**: `portainer-pull`
   - **Scopes**: Check `read_registry`
4. Copy the **username** and **token** - you won't see the token again.

## Creating a Personal Access Token (Alternative)

```bash
# In GitLab UI: User Settings > Access Tokens

# Create a token with scope: read_registry
# Token acts as password, your GitLab username as username
```

## Adding GitLab Registry to Portainer

1. Go to **Settings > Registries** and click **Add registry**.
2. Select **GitLab** or **Custom registry**.
3. Enter:
   - **Registry URL**: `registry.gitlab.com`
   - **Username**: Your deploy token username (e.g., `gitlab+deploy-token-123`)
   - **Password**: Your deploy token value
4. Click **Add registry**.

## For Self-Hosted GitLab

If you run GitLab on your own server, use your instance's registry URL:

```text
registry.yourcompany.com
```

The setup is identical - create a deploy token in your self-hosted GitLab and use it in Portainer.

## Using GitLab Registry Images in a Stack

```yaml
version: "3.8"

services:
  app:
    # Portainer uses the stored GitLab registry credentials
    image: registry.gitlab.com/mygroup/myproject/app:latest
    deploy:
      replicas: 2
```

## Testing GitLab Registry Access

```bash
# Test login to GitLab registry via CLI
docker login registry.gitlab.com \
  -u gitlab+deploy-token-123 \
  -p <your-deploy-token>

# Pull an image to verify
docker pull registry.gitlab.com/mygroup/myproject/app:latest
```

## CI/CD Integration

In your `.gitlab-ci.yml`, push images that Portainer can then pull:

```yaml
# .gitlab-ci.yml snippet for building and pushing to GitLab registry
build:
  stage: build
  image: docker:24
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

## Conclusion

GitLab's built-in registry and deploy tokens make it straightforward to integrate with Portainer. Use deploy tokens (not personal access tokens) for service accounts to limit exposure and enable easy rotation.
