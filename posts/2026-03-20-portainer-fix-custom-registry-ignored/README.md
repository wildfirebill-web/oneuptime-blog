# How to Fix "Custom Registry Credentials Ignored" in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Registry, Troubleshooting, Authentication, DevOps

Description: Learn how to diagnose and fix the common Portainer issue where custom registry credentials are ignored during image pulls.

## Introduction

A common frustration with Portainer is configuring a custom private registry and then finding that images fail to pull because the credentials are seemingly ignored. This can happen for several reasons: URL mismatch between the registry configuration and the image reference, Docker credentials caching, or Portainer configuration issues. This guide covers systematic diagnosis and fixes.

## Common Symptoms

```
# Pull fails even though registry is configured
Error: pull access denied for registry.company.com/myimage,
repository does not exist or may require 'docker login'

# Or authentication still fails
Error: unauthorized: authentication required

# Or wrong registry used
Pulling from docker.io instead of registry.company.com
```

## Root Cause Analysis

### Cause 1: URL Mismatch

The most common cause — the URL in Portainer's registry config doesn't match the image URL prefix.

```
Portainer registry URL:  registry.company.com
Image in stack:          https://registry.company.com/myimage:latest  ← Has https://

Portainer registry URL:  registry.company.com:5000
Image in stack:          registry.company.com/myimage:latest  ← Missing port
```

Portainer matches credentials to images by comparing the URL prefix exactly.

**Fix:**

```yaml
# If Portainer registry URL is: registry.company.com
# Image in Compose must be:
services:
  app:
    image: registry.company.com/myimage:latest   # No https://, correct port
```

### Cause 2: Docker Credentials Cache

Docker caches credentials in `~/.docker/config.json`. If there are cached but incorrect credentials for a registry, they override Portainer's stored ones.

```bash
# Check Docker credential cache
cat ~/.docker/config.json

# Remove cached credentials for a specific registry
docker logout registry.company.com

# Or edit the file directly to remove the entry
```

### Cause 3: Portainer Agent vs Direct Connection

For remote environments using the Portainer Agent, credentials are passed from the Portainer server to the agent. If the agent can't reach the registry, image pulls fail.

```bash
# Test connectivity from the Docker host (where agent runs), not from Portainer server
curl -u user:pass https://registry.company.com/v2/_catalog
```

### Cause 4: Portainer CE Limitations

In Portainer CE, registry credentials for Kubernetes are handled differently than for Docker. For Kubernetes environments, you need to create Kubernetes image pull secrets separately.

## Step 1: Verify the Registry URL Configuration

```bash
# In Portainer, go to Registries and check the URL
# The URL should match EXACTLY the prefix used in your image names

# Example image: harbor.company.com/prod/myapp:latest
# Registry URL should be: harbor.company.com  (NOT https://harbor.company.com)
```

## Step 2: Test Credentials from the Docker Host

```bash
# SSH into the Docker host
ssh user@docker-host

# Test the exact credentials configured in Portainer
docker login harbor.company.com \
  --username portainer-user \
  --password mypassword

# If this fails, the credentials are wrong — fix them in Portainer
```

## Step 3: Clear Docker Credential Cache

```bash
# View current credentials cache
cat /root/.docker/config.json

# Remove stale credentials
docker logout harbor.company.com

# Verify the cache entry is removed
cat /root/.docker/config.json
```

## Step 4: Verify Portainer Version and Registry Support

```bash
# Portainer versions have different registry handling
docker exec portainer portainer --version
```

Known issues:
- Some older Portainer CE versions have bugs with credential passing to Swarm agents
- Portainer 2.x handles credentials differently than 1.x

## Step 5: Use Docker Config Secret (Kubernetes)

For Kubernetes environments, Portainer needs a Kubernetes secret, not just the registry entry:

```bash
# Create image pull secret in the target namespace
kubectl create secret docker-registry regcred \
  --docker-server=registry.company.com \
  --docker-username=portainer-user \
  --docker-password=mypassword \
  --namespace=production

# Reference in deployment
```

```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: registry.company.com/myapp:latest
```

## Step 6: Force Re-authentication

Sometimes resetting the registry configuration in Portainer helps:

1. Go to **Registries**
2. Delete the failing registry
3. Re-add it with fresh credentials
4. Test a deployment immediately

## Step 7: Check Portainer Agent Version

If using remote agents, ensure the agent and server versions match:

```bash
# Check agent version
docker exec portainer_agent portainer-agent --version

# Agent and server should be the same version
# Mismatched versions can cause credential passing failures
```

## Step 8: Enable Debug Logging

For deeper investigation:

```bash
# Restart Portainer with debug logging
docker run -d \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --log-level DEBUG

# Check logs
docker logs -f portainer | grep -i "registry\|credential\|auth"
```

## Step 9: Manual Docker Login as Workaround

If Portainer credentials still aren't working, manually log into the registry on the Docker host:

```bash
# This creates credentials that Docker daemon uses
docker login registry.company.com \
  --username user \
  --password password

# The credentials persist in /root/.docker/config.json
# Docker will use these when Portainer triggers a pull
```

This is a workaround, not a permanent fix. The root cause should still be identified.

## Prevention Checklist

```
[ ] Registry URL in Portainer matches image URL prefix exactly
[ ] No https:// prefix in the registry URL
[ ] Port number included if non-standard (e.g., :5000)
[ ] Credentials tested directly on the Docker host
[ ] Docker credential cache checked for conflicts
[ ] For Kubernetes: image pull secrets created in namespaces
[ ] Portainer agent and server versions match
```

## Conclusion

The "custom registry credentials ignored" issue almost always comes down to a URL mismatch or credential caching problem. Verify that the Portainer registry URL exactly matches the prefix in your image names, test credentials directly on the Docker host, and clear any conflicting cached credentials. For Kubernetes deployments, remember that image pull secrets must be created as Kubernetes resources, not just as Portainer registry configurations.
