# How to Fix "Custom Registry Credentials Ignored" in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Container Registry, Troubleshooting, Docker, DevOps

Description: Learn how to diagnose and fix issues where Portainer ignores stored custom registry credentials when pulling images.

## The Problem

You've added a custom private registry to Portainer with credentials, but when deploying containers or stacks, Portainer still fails with `unauthorized` errors — as if the credentials aren't being used.

## Common Causes

1. The registry is not enabled for the current environment.
2. The image URL doesn't match the registry URL exactly.
3. The stack references a full registry URL but Portainer matches only by hostname.
4. The registry was added after the environment was created (requires re-assignment).
5. Credentials are correct but the registry uses an insecure connection.

## Fix 1: Ensure the Registry Is Enabled for the Environment

Registries in Portainer are global, but must be explicitly assigned to each environment:

1. Go to **Environments** and select your environment.
2. Click **Edit**.
3. Scroll to **Registries** and check that your custom registry is toggled **On**.
4. Click **Update environment**.

## Fix 2: Match the Image URL to the Registry URL

The registry URL in Portainer must exactly match the hostname in your image reference:

```yaml
# If your registry is registered as: registry.mycompany.com
# Your image MUST reference the same hostname exactly:
image: registry.mycompany.com/myteam/myapp:latest

# NOT:
image: registry.mycompany.com:443/myteam/myapp:latest  # Port mismatch
image: https://registry.mycompany.com/myteam/myapp:latest  # Protocol prefix
```

## Fix 3: Check for Port Mismatches

```bash
# Portainer registry URL (what's stored in Portainer):
registry.mycompany.com:5000

# Image reference in your stack:
registry.mycompany.com:5000/myapp:latest  # Must include the port
```

## Fix 4: Re-Test Credentials

```bash
# Manually test the credentials Portainer is using
docker login registry.mycompany.com \
  -u <your-username> \
  -p <your-password>

# If this fails, the credentials themselves are wrong
```

## Fix 5: Handle Insecure Registries

If your registry doesn't use HTTPS, Docker rejects it unless configured as insecure:

```json
// /etc/docker/daemon.json - add to every Swarm/Docker node
{
  "insecure-registries": ["registry.mycompany.com:5000"]
}
```

```bash
sudo systemctl restart docker
```

## Fix 6: Verify via Portainer API

```bash
# List registries to check their configuration
curl -H "Authorization: Bearer $PORTAINER_TOKEN" \
  http://localhost:9000/api/registries | jq '.[] | {id, name, url, authentication}'
```

## Conclusion

The most common cause of ignored credentials in Portainer is either the registry not being assigned to the environment or an exact URL mismatch. Always verify both the registry assignment and that your image references match the stored registry hostname precisely.
