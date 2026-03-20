# How to Add a Custom Private Registry to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Private Registry, Docker Registry, Container Management, DevOps

Description: Learn how to connect a self-hosted Docker registry to Portainer so you can pull and push private images.

## Overview

Many teams run their own private Docker registry using the official `registry:2` image. Portainer supports connecting to any custom registry that speaks the Docker Registry HTTP API v2.

## Setting Up a Local Registry

Before adding it to Portainer, ensure your registry is running:

```bash
# Run a basic private registry on port 5000
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /registry-data:/var/lib/registry \
  registry:2
```

## Adding a Custom Registry to Portainer

1. Log in to Portainer as an administrator.
2. Navigate to **Settings > Registries**.
3. Click **Add registry**.
4. Select **Custom registry**.
5. Fill in the fields:
   - **Name**: A friendly name (e.g., "My Private Registry")
   - **Registry URL**: The URL to your registry (e.g., `registry.mycompany.com:5000`)
   - **Authentication**: Enable and enter username/password if your registry requires auth
6. Click **Add registry**.

## Running a Registry with Authentication

For production, use a registry with basic authentication:

```bash
# Generate an htpasswd file for basic auth
docker run --rm \
  --entrypoint htpasswd \
  httpd:2 -Bbn myuser mypassword > /auth/htpasswd

# Run the registry with authentication
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /registry-data:/var/lib/registry \
  -v /auth:/auth \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

## Pushing Images to Your Private Registry

```bash
# Tag a local image for your private registry
docker tag nginx:alpine registry.mycompany.com:5000/nginx:alpine

# Push the tagged image
docker push registry.mycompany.com:5000/nginx:alpine
```

## Using the Registry in Portainer

After registering, your custom registry appears in the **Registry** dropdown when deploying containers or stacks:

```yaml
version: "3.8"

services:
  web:
    # Reference images from your custom registry
    image: registry.mycompany.com:5000/nginx:alpine
```

## Insecure Registries (HTTP)

If your registry uses HTTP (not HTTPS), add it to Docker's insecure registries list on each node:

```json
// /etc/docker/daemon.json
{
  "insecure-registries": ["registry.mycompany.com:5000"]
}
```

## Conclusion

Portainer makes it easy to add self-hosted private registries as centrally managed sources. Always use HTTPS and authentication for production registries to keep your images secure.
