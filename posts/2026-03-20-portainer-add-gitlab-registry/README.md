# How to Add GitLab Container Registry to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, GitLab, Registry, CI/CD, DevOps

Description: Learn how to configure GitLab Container Registry in Portainer to deploy images built by GitLab CI/CD pipelines.

## Introduction

GitLab's built-in Container Registry provides a convenient way to store images built by GitLab CI/CD pipelines. Portainer can be configured to pull from GitLab Container Registry using deploy tokens or personal access tokens, enabling a seamless GitOps workflow where GitLab builds images and Portainer deploys them.

## Prerequisites

- Portainer CE or BE installed
- GitLab instance (gitlab.com or self-hosted) with Container Registry enabled
- A project with container images
- GitLab account with appropriate permissions

## GitLab Registry URL Formats

| GitLab Type | Registry URL |
|-------------|-------------|
| gitlab.com | `registry.gitlab.com` |
| Self-hosted | `registry.yourdomain.com` or `gitlab.yourdomain.com:5050` |

Image paths follow the pattern:

```
registry.gitlab.com/{namespace}/{project-name}/{image-name}:{tag}
# Example:
registry.gitlab.com/myorg/myapp/backend:latest
registry.gitlab.com/myorg/myapp/frontend:v2.0
```

## Step 1: Create a Deploy Token (Recommended for Production)

Deploy tokens are project or group-level tokens specifically designed for deployment use cases:

1. In GitLab, navigate to your project or group
2. Go to **Settings → Repository → Deploy tokens**
3. Click **Add a deploy token**
4. Fill in:
   ```
   Name:       portainer-pull
   Expires at: (set 1 year from now)
   Username:   portainer (custom username)
   Scopes:     [x] read_registry
   ```
5. Click **Create deploy token**
6. Copy the **username** and **token** (shown once)

## Step 2: Create a Personal Access Token (Alternative)

For personal projects or testing:

1. Go to **User Settings → Access Tokens**
2. Click **Add a personal access token**
3. Configure:
   ```
   Token name:     portainer-registry
   Expiration:     (set future date)
   Scopes:         [x] read_registry
   ```
4. Click **Create personal access token**
5. Copy the generated token

## Step 3: Add GitLab Registry in Portainer

1. Go to **Registries** in Portainer
2. Click **+ Add registry**
3. Select **GitLab** (or **Custom registry** if GitLab option is not available)

### For gitlab.com

```
Registry type:  GitLab
URL:           registry.gitlab.com
Username:      portainer          (deploy token username)
Password:      gldt-xxxxx...     (deploy token value)
```

### For Self-Hosted GitLab

```
Registry type:  Custom registry
URL:           registry.gitlab.yourdomain.com
Username:      portainer          (or your GitLab username for PAT)
Password:      gldt-xxxxx...     (deploy token or PAT)
```

4. Click **Add registry**

## Step 4: Configure GitLab CI to Build and Push Images

Set up your `.gitlab-ci.yml` to build and push images automatically:

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

trigger-portainer-deploy:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache curl
  script:
    # Trigger Portainer webhook to update the service
    - |
      curl -X POST "$PORTAINER_WEBHOOK_URL" \
        --fail \
        --max-time 30
  only:
    - main
```

## Step 5: Use GitLab Registry Images in Portainer Stacks

```yaml
version: "3.8"

services:
  backend:
    image: registry.gitlab.com/myorg/myapp/backend:latest
    # Portainer uses stored GitLab registry credentials

  frontend:
    image: registry.gitlab.com/myorg/myapp/frontend:v2.0
    ports:
      - "80:80"
```

## Step 6: Enable the Portainer Webhook in GitLab

For automatic deployments when CI builds a new image:

1. Get the Portainer service/stack webhook URL from Portainer
2. In GitLab, set it as a CI/CD variable:
   - **Project → Settings → CI/CD → Variables**
   - Add `PORTAINER_WEBHOOK_URL` as a masked variable

3. Add the deploy step to your GitLab CI pipeline (see Step 4 above)

## Step 7: Use Group Deploy Tokens for Multiple Projects

For organizations with multiple GitLab projects deploying to the same Portainer:

1. Create a **Group Deploy Token** at the group level
2. Grant `read_registry` scope
3. Use the same token in Portainer for all projects under that group

```
Group URL:   registry.gitlab.com/myorg/{any-project-in-group}/{image}:tag
```

## Troubleshooting

### Authentication Failed

```
Error: unauthorized: HTTP Basic: Access denied
```

- Verify the deploy token or PAT is valid and not expired
- Confirm the token has `read_registry` scope
- For PATs, ensure the user has at least Reporter access to the project

### Registry Not Enabled

If the registry URL returns 404, the Container Registry may be disabled for your project:

1. Go to **Project Settings → General → Visibility, project features, permissions**
2. Enable **Container Registry**

### Self-Signed Certificate

For self-hosted GitLab with self-signed TLS:

```bash
# Add certificate to Docker hosts
sudo cp gitlab.crt /etc/docker/certs.d/registry.gitlab.yourdomain.com/ca.crt
sudo systemctl restart docker
```

## Conclusion

Integrating GitLab Container Registry with Portainer closes the loop on your CI/CD pipeline — GitLab builds and stores images, and Portainer deploys them. Using deploy tokens with minimal scope (`read_registry` only) follows security best practices. Combined with GitLab CI webhooks triggering Portainer deployments, you have a complete automated deployment workflow.
