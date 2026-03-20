# How to Troubleshoot Registry Authentication Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Container Registry, Troubleshooting, Authentication, Docker

Description: Learn how to diagnose and fix registry authentication failures when pulling images in Portainer.

## Common Error Messages

| Error | Likely Cause |
|-------|-------------|
| `unauthorized: authentication required` | Wrong credentials or no credentials stored |
| `denied: requested access to resource is denied` | Credentials valid but insufficient permissions |
| `no such host` | Wrong registry URL |
| `x509: certificate signed by unknown authority` | Self-signed or expired TLS certificate |
| `toomanyrequests` | Docker Hub rate limit exceeded |

## Step 1: Test Credentials Manually

Before debugging Portainer, verify the credentials work directly:

```bash
# Test Docker Hub login
docker login docker.io -u myuser -p mypassword

# Test a custom registry
docker login registry.mycompany.com -u myuser -p mypassword

# Test pulling an image directly
docker pull registry.mycompany.com/myimage:latest
```

## Step 2: Check Registry URL Format

Common mistakes in registry URLs:

```bash
# Correct formats
docker.io                           # Docker Hub
ghcr.io                             # GitHub Container Registry
123456789.dkr.ecr.us-east-1.amazonaws.com  # AWS ECR
registry.mycompany.com              # Custom registry

# Incorrect - don't include protocol or trailing slash
https://registry.mycompany.com/     # Wrong
registry.mycompany.com/             # Wrong (trailing slash)
```

## Step 3: Handle Self-Signed Certificates

If your registry uses a self-signed or private CA certificate:

```bash
# On each Docker host, copy the certificate
sudo mkdir -p /etc/docker/certs.d/registry.mycompany.com
sudo cp ca.crt /etc/docker/certs.d/registry.mycompany.com/ca.crt

# Restart Docker to apply
sudo systemctl restart docker
```

## Step 4: Check Token Expiry (ECR)

AWS ECR tokens expire after 12 hours. If pulls suddenly fail, the token likely expired:

```bash
# Re-authenticate with ECR and update Portainer
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com
```

## Step 5: Verify Registry Is Assigned to the Environment

Even if a registry is added to Portainer, it must be enabled per environment:

1. Go to **Environments**, select your environment.
2. Scroll to **Registries** and verify your registry is enabled.

## Step 6: Check Portainer Logs

```bash
# Check Portainer container logs for authentication errors
docker logs portainer 2>&1 | grep -i "registry\|auth\|unauthorized" | tail -20
```

## Step 7: Diagnose Rate Limiting

```bash
# Check Docker Hub rate limit status
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)

curl -I --head \
  -H "Authorization: Bearer $TOKEN" \
  https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest
# Look for: RateLimit-Remaining header
```

## Conclusion

Registry authentication issues in Portainer usually fall into a few categories: wrong credentials, certificate problems, expired tokens, or misconfigured environment assignments. Work through each step systematically to isolate the root cause.
