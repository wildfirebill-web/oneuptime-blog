# How to Add Docker Hub Credentials to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Hub, Registry, Authentication, Container Management

Description: Learn how to add Docker Hub credentials to Portainer to pull private images and avoid rate limiting.

## Why Add Docker Hub Credentials?

Docker Hub enforces rate limits on anonymous pulls (100 pulls per 6 hours for unauthenticated users, 200 for free accounts). Authenticated pulls use your account's higher limits. Additionally, private Docker Hub repositories require credentials to pull images.

## Steps to Add Docker Hub Credentials

1. Log in to Portainer as an administrator.
2. Go to **Settings** in the left sidebar.
3. Click **Registries**.
4. Click **Add registry**.
5. Select **DockerHub** as the registry provider.
6. Enter your Docker Hub **username** and **password** (or access token).
7. Click **Add registry**.

## Using Access Tokens Instead of Passwords

Docker Hub supports access tokens as a more secure alternative to passwords:

1. Log in to [hub.docker.com](https://hub.docker.com).
2. Go to **Account Settings > Security**.
3. Click **New Access Token**, give it a name, and copy the token.
4. Use this token as the **password** field in Portainer.

```bash
# Test your credentials via CLI before adding to Portainer

docker login -u your-username
# Enter your access token when prompted for password
```

## Assigning Credentials to Environments

After adding the registry, you need to make it available in your environments:

1. Go to **Environments** and select your environment.
2. Scroll to **Registries** and enable the Docker Hub registry for that environment.

## Using Credentials When Pulling Images

When deploying a container or stack in Portainer, the registry credentials are automatically used when pulling images from Docker Hub. For stacks, ensure your image reference matches the registry:

```yaml
version: "3.8"

services:
  app:
    # Portainer will use the stored Docker Hub credentials to pull this image
    image: your-dockerhub-username/private-image:latest
```

## Verifying Credentials

```bash
# Verify your Docker Hub login works from the CLI
docker login docker.io -u your-username -p your-access-token

# Test pulling a private image
docker pull your-username/private-image:latest
```

## Conclusion

Adding Docker Hub credentials to Portainer ensures private images are accessible and prevents anonymous rate limit errors. Use access tokens rather than passwords for better security and easier rotation.
