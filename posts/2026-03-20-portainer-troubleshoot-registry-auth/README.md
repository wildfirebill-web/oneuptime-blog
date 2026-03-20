# How to Troubleshoot Registry Authentication Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Registry, Authentication, Troubleshooting, DevOps

Description: A systematic guide to diagnosing and fixing container registry authentication failures in Portainer.

## Introduction

Registry authentication failures are some of the most common errors when deploying containers through Portainer. The errors can be cryptic and the causes range from expired tokens to network issues to configuration mistakes. This guide provides a systematic approach to diagnosing and fixing registry authentication problems.

## Common Error Messages

```
# Docker Hub authentication failure
Error: pull access denied for myorg/myimage, repository does not exist or may require 'docker login'

# Private registry auth failure
Error: Get https://registry.company.com/v2/: dial tcp: connect: connection refused

# Invalid credentials
Error: unauthorized: incorrect username or password

# Token expired (ECR)
Error: no basic auth credentials

# SSL certificate issue
Error: x509: certificate signed by unknown authority

# Rate limit exceeded
Error: toomanyrequests: You have reached your pull rate limit
```

## Step 1: Identify the Registry Type

First, determine which registry is failing:

```bash
# Check the image name to identify the registry
docker pull myorg/myimage:latest          # Docker Hub
docker pull registry.company.com/...      # Private custom registry
docker pull 123456.dkr.ecr.*.amazonaws.com/... # AWS ECR
docker pull myregistry.azurecr.io/...    # Azure ACR
docker pull gcr.io/... or *.pkg.dev/...  # Google Cloud
docker pull ghcr.io/...                  # GitHub GHCR
docker pull registry.gitlab.com/...      # GitLab
```

## Step 2: Test Registry Connectivity

```bash
# Basic connectivity test
curl -I https://registry.company.com/v2/

# Expected responses:
# 200 OK: Registry is up and auth is not required
# 401 Unauthorized: Registry is up, auth is required (normal)
# 000 or connection refused: Registry is down or wrong URL
# 403 Forbidden: Auth failed
```

## Step 3: Test Credentials Directly

```bash
# Test credentials without Docker
curl -u "username:password" https://registry.company.com/v2/_catalog

# Docker Hub
curl -u "username:password" https://hub.docker.com/v2/repositories/

# AWS ECR - first get a token
TOKEN=$(aws ecr get-login-password --region us-east-1)
curl -u "AWS:$TOKEN" https://123456.dkr.ecr.us-east-1.amazonaws.com/v2/_catalog

# Azure ACR
curl -u "username:password" https://myregistry.azurecr.io/v2/_catalog
```

## Step 4: Test with Docker Login

```bash
# Test the exact credentials Portainer would use
docker login registry.company.com \
  --username portainer-user \
  --password mypassword

# If this fails, the issue is in your credentials, not Portainer
```

## Step 5: Check Credentials in Portainer

1. Go to **Registries** in Portainer
2. Click **Edit** on the failing registry
3. Re-enter the password/token (Portainer shows asterisks, not the actual value)
4. Click **Save**

Common mistakes to check:

- Extra spaces before/after the username or password
- Password contains special characters that need escaping
- Wrong username format (e.g., GitLab robot accounts need `robot$` prefix)
- Token vs password mixed up

## Step 6: Diagnose SSL/TLS Issues

```bash
# Check SSL certificate chain
openssl s_client -connect registry.company.com:443 \
  -servername registry.company.com

# Check certificate expiry
echo | openssl s_client -connect registry.company.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# If self-signed: add CA cert to Docker
sudo mkdir -p /etc/docker/certs.d/registry.company.com
sudo cp ca.crt /etc/docker/certs.d/registry.company.com/ca.crt
sudo systemctl restart docker
```

## Step 7: Diagnose Token Expiration (ECR/ACR)

ECR tokens expire every 12 hours:

```bash
# Check if ECR token is still valid
aws ecr describe-repositories --region us-east-1

# If this works but Docker fails, the stored token is expired
# Refresh: re-run the ECR token and update Portainer registry credentials

# Azure ACR - check if service principal is valid
az ad sp show --id <service-principal-id>
# Check "accountEnabled" and "appOwnerOrganizationId"
```

## Step 8: Check DNS Resolution

```bash
# Verify DNS resolves correctly
nslookup registry.company.com
dig registry.company.com

# From inside a Portainer container
docker exec -it portainer nslookup registry.company.com
```

If DNS fails from the Docker host, images cannot be pulled.

## Step 9: Check Firewall and Network Access

```bash
# Test TCP connectivity to registry port
nc -zv registry.company.com 443

# From the Docker host (not just your workstation)
# Docker pulls happen from the Docker daemon, not from your client machine
curl --max-time 5 https://registry.company.com/v2/
```

## Step 10: Check Docker Daemon Configuration

```bash
# Verify insecure registries are configured (for HTTP registries)
docker info | grep -A5 "Insecure Registries"

# Check /etc/docker/daemon.json
cat /etc/docker/daemon.json
```

For an HTTP registry, ensure:

```json
{
  "insecure-registries": ["registry.company.com:5000"]
}
```

## Step 11: View Portainer Agent/Server Logs

```bash
# Check Portainer server logs for auth-related errors
docker logs portainer 2>&1 | grep -i "registry\|auth\|unauthorized" | tail -20
```

## Step 12: Verify Image Name Format

Common image name mistakes:

```bash
# Correct formats:
myimage:latest                                           # Docker Hub, official
myorg/myimage:v1.0                                       # Docker Hub, user/org image
registry.company.com/project/myimage:latest              # Private registry
123456.dkr.ecr.us-east-1.amazonaws.com/myimage:latest   # AWS ECR (full URL)

# Wrong formats (common mistakes):
https://registry.company.com/myimage    # Don't include https://
registry.company.com//myimage           # Double slash
```

## Diagnostic Checklist

```
[ ] Registry URL is correct (no trailing slash, correct port)
[ ] Username is correct (check for robot$ prefix for Harbor/GitLab)
[ ] Password/token is current (not expired)
[ ] Network connectivity to registry from Docker host
[ ] DNS resolves correctly
[ ] SSL certificate is valid or CA cert is installed
[ ] For HTTP registries: insecure-registries is configured
[ ] Image name format is correct
[ ] Repository exists and user has read access
```

## Conclusion

Registry authentication failures follow predictable patterns. Work through the diagnostic steps systematically: verify connectivity, test credentials directly, check token expiration, and ensure SSL certificates are properly configured. Most issues fall into a handful of categories: expired tokens, wrong credentials, network connectivity, or SSL certificate problems. With this guide, you can quickly identify and resolve the root cause.
