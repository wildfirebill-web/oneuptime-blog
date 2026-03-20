# How to Restrict Public Repository Usage in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Registry, Docker, Policy

Description: Learn how to restrict Portainer users from pulling container images from public registries, enforcing the use of approved private registries only.

## Introduction

Unrestricted access to public image registries (Docker Hub, GitHub Container Registry, etc.) can introduce unvetted, potentially malicious images into your environment. Portainer allows administrators to restrict users to approved registries only, enforcing image provenance policies.

## Why Restrict Public Registries?

- **Supply chain security**: Prevent use of unvetted public images
- **Compliance**: Many frameworks require using internal/approved registries
- **Stability**: Public image tags can change unexpectedly
- **Cost control**: Avoid unexpected registry pull fees
- **Security scanning**: Only deploy images that have passed your security pipeline

## Step 1: Configure Approved Registries

Before restricting public access, add your approved registries:

1. Go to Portainer **Registries**.
2. Add your private registry (Harbor, ECR, GHCR with org scope, etc.).
3. Configure authentication.

```bash
TOKEN="your-admin-token"
PORTAINER_URL="https://portainer.example.com"

# Add your private registry
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/registries" \
  -d '{
    "Name": "Company Registry",
    "Type": 6,
    "URL": "registry.company.com",
    "Authentication": true,
    "Username": "portainer-svc",
    "Password": "registry-password"
  }' | jq '{id: .Id, name: .Name}'
```

## Step 2: Enable Registry Restrictions for Users

### Via Portainer UI (Global Settings)

1. Go to **Settings** → **Registries**.
2. Enable **Restrict users from pulling Docker Hub images**.
3. Optionally enable **Restrict public images** (require only authenticated registries).
4. Save settings.

### Via Environment-Specific Settings

For environment-level restrictions:

1. Select your Docker environment.
2. Go to **Settings**.
3. Find the registry restriction options.
4. Check **Restrict users to use defined registries only**.
5. Select which registries are allowed in this environment.

## Step 3: Configure via API

```bash
# Restrict to approved registries only in global settings
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/settings" \
  -d '{
    "allowPrivilegedModeForRegularUsers": false,
    "enableHostManagementFeatures": false
  }' | jq .
```

## Step 4: Mirror Public Images to Private Registry

When restricting public registries, mirror commonly needed images to your private registry:

```bash
#!/bin/bash
# mirror-images.sh — Mirror approved public images to private registry

PRIVATE_REGISTRY="registry.company.com"
APPROVED_IMAGES=(
  "nginx:1.25"
  "nginx:1.24"
  "postgres:15-alpine"
  "postgres:14-alpine"
  "redis:7-alpine"
  "redis:6-alpine"
  "node:20-alpine"
  "node:18-alpine"
  "python:3.11-alpine"
  "alpine:3.19"
  "ubuntu:22.04"
)

for IMAGE in "${APPROVED_IMAGES[@]}"; do
  PRIVATE_IMAGE="${PRIVATE_REGISTRY}/mirrors/${IMAGE}"

  echo "Mirroring: $IMAGE → $PRIVATE_IMAGE"

  # Pull from public registry
  docker pull "$IMAGE"

  # Tag for private registry
  docker tag "$IMAGE" "$PRIVATE_IMAGE"

  # Push to private registry
  docker push "$PRIVATE_IMAGE"

  echo "Done: $PRIVATE_IMAGE"
done

echo "All images mirrored to $PRIVATE_REGISTRY/mirrors/"
```

## Step 5: Automated Image Scanning Pipeline

For a complete supply chain security solution, scan images before mirroring:

```bash
#!/bin/bash
# scan-and-mirror.sh — Scan public images before adding to private registry

PRIVATE_REGISTRY="registry.company.com"
IMAGE=$1  # e.g., "nginx:1.25"

echo "Scanning $IMAGE for vulnerabilities..."

# Pull and scan with Trivy
docker pull "$IMAGE"
SCAN_RESULT=$(docker run --rm aquasec/trivy:latest image --severity HIGH,CRITICAL "$IMAGE")
HIGH_COUNT=$(echo "$SCAN_RESULT" | grep -c "HIGH" || true)
CRITICAL_COUNT=$(echo "$SCAN_RESULT" | grep -c "CRITICAL" || true)

echo "Scan results: $HIGH_COUNT HIGH, $CRITICAL_COUNT CRITICAL vulnerabilities"

if [ "$CRITICAL_COUNT" -gt 0 ]; then
  echo "REJECTED: Image has CRITICAL vulnerabilities"
  exit 1
fi

if [ "$HIGH_COUNT" -gt 10 ]; then
  echo "REJECTED: Too many HIGH vulnerabilities ($HIGH_COUNT > 10 threshold)"
  exit 1
fi

echo "APPROVED: Mirroring to private registry..."
PRIVATE_IMAGE="${PRIVATE_REGISTRY}/approved/${IMAGE}"
docker tag "$IMAGE" "$PRIVATE_IMAGE"
docker push "$PRIVATE_IMAGE"

echo "Available at: $PRIVATE_IMAGE"
```

## Step 6: Notification on Policy Violation

Configure alerts when users attempt to use unapproved registries:

```bash
# Monitor Portainer logs for registry denial events
docker logs portainer 2>&1 | grep -i "registry\|unauthorized\|denied" | tail -20

# Set up a log monitoring script
#!/bin/bash
# monitor-registry-violations.sh

PORTAINER_LOGS=$(docker logs portainer 2>&1 | tail -100)
VIOLATIONS=$(echo "$PORTAINER_LOGS" | grep -i "registry.*denied\|unauthorized.*registry")

if [ -n "$VIOLATIONS" ]; then
  echo "Registry policy violations detected:"
  echo "$VIOLATIONS"
  # Send to Slack, PagerDuty, email, etc.
fi
```

## Step 7: Exception Process

Document an exception process for approved public images:

```yaml
# image-approval-request.yml — Template for requesting public image approval
request:
  requester: "developer@company.com"
  image: "docker.io/library/nginx:1.25"
  reason: "Official NGINX base image for web server"
  scan_results: "0 CRITICAL, 2 HIGH (accepted risk - no fixes available)"
  approved_by: ""
  approval_date: ""
  mirror_path: "registry.company.com/approved/nginx:1.25"
  review_date: "2026-09-20"  # 6 months from approval
```

## Conclusion

Restricting public repository usage in Portainer creates a controlled image supply chain. Add your approved private registries, mirror commonly needed images with security scanning, and configure Portainer to block access to unapproved public registries. Document an exception and approval process for new images, and schedule regular reviews to remove outdated approved images from your mirror.
