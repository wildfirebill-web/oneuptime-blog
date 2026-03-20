# How to Set Up Docker Security Policies in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Docker, Policies, Hardening

Description: Learn how to configure comprehensive Docker security policies in Portainer to enforce container security standards across all environments and user groups.

## Introduction

Portainer provides a centralized way to enforce Docker security policies across your container infrastructure. By configuring security settings at the environment level, you can ensure all containers — regardless of who deploys them — adhere to your organization's security standards.

## Prerequisites

- Portainer CE or BE with admin access
- Docker environments connected to Portainer
- Security requirements documented for your organization

## Available Security Policies in Portainer

| Policy | Protection |
|--------|-----------|
| Disable bind mounts | Prevents host filesystem access |
| Disable privileged mode | Prevents host kernel access |
| Disable host PID | Prevents process namespace sharing |
| Disable host network | Prevents network namespace sharing |
| Disable device mapping | Prevents host device access |
| Disable sysctl settings | Prevents kernel parameter changes |
| Restrict public registries | Enforce approved image sources |
| Require image signing | Content trust enforcement |

## Step 1: Configure Security Policies in the UI

For each environment:

1. Log into Portainer as admin.
2. Select your environment.
3. Click **Settings** (gear icon).
4. Scroll to **Security**.
5. Enable the policies you want to enforce.
6. Click **Save**.

## Step 2: Configure All Policies via API

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
ENDPOINT_ID=1

# Apply comprehensive security policy
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/settings" \
  -d '{
    "disableBindMountsForRegularUsers": true,
    "disablePrivilegeModeForRegularUsers": true,
    "disableHostPidForRegularUsers": true,
    "disableHostNetworkModeForRegularUsers": true,
    "disableDeviceMappingForRegularUsers": true,
    "disableSysctlSettingForRegularUsers": true,
    "enableHostManagementFeatures": false
  }' | jq .
```

## Step 3: Restrict Image Pulling to Approved Registries

In Portainer global settings:

1. Go to **Settings** → **Registries**.
2. Add only approved registries.
3. Under **Security** settings, configure **Restrict public images**.

Via API:

```bash
# Configure registry restrictions
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/settings" \
  -d '{
    "blackListedLabels": [],
    "allowPrivilegedModeForRegularUsers": false,
    "allowVolumeBrowserForRegularUsers": false,
    "enableHostManagementFeatures": false
  }' | jq .
```

## Step 4: Enforce Secure Container Templates

Create approved App Templates that already include security settings:

```json
// portainer-templates.json — Secure application templates
[
  {
    "type": 1,
    "title": "Secure Web App",
    "description": "Pre-configured with security best practices",
    "categories": ["Web"],
    "platform": "linux",
    "image": "nginx:1.25",
    "ports": ["80/tcp"],
    "env": [],
    "volumes": [],
    "privileged": false,
    "note": "Production-hardened configuration. No privileged mode, no bind mounts."
  }
]
```

Upload templates in Portainer:
1. Go to **Settings** → **App Templates**.
2. Enter the URL to your templates JSON file.
3. Save.

## Step 5: Implement Docker Daemon Security Defaults

On the Docker host itself, configure secure defaults:

```json
// /etc/docker/daemon.json — Docker daemon security settings
{
  "icc": false,                          // Disable inter-container communication by default
  "no-new-privileges": true,             // Prevent privilege escalation in containers
  "userns-remap": "default",             // Enable user namespace remapping
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,               // Use iptables instead of userland proxy
  "seccomp-profile": "/etc/docker/seccomp-profile.json"  // Custom seccomp profile
}
```

```bash
# Apply daemon configuration
sudo systemctl restart docker

# Verify
docker info | grep -E "(Security|Namespace|Seccomp)"
```

## Step 6: Create a Security Audit Script

```bash
#!/bin/bash
# security-audit.sh — Audit running containers for security violations

PORTAINER_URL="https://portainer.example.com"
TOKEN="your-token"
ENDPOINT_ID=1

echo "=== Container Security Audit ==="
echo ""

CONTAINERS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json?all=false")

VIOLATIONS=0

echo $CONTAINERS | jq -c '.[]' | while read -r CONTAINER; do
  NAME=$(echo $CONTAINER | jq -r '.Names[0]' | sed 's/^\///')
  ID=$(echo $CONTAINER | jq -r '.Id')

  # Inspect each container
  INSPECT=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${ID}/json")

  PRIVILEGED=$(echo $INSPECT | jq -r '.HostConfig.Privileged')
  PID_MODE=$(echo $INSPECT | jq -r '.HostConfig.PidMode')
  NET_MODE=$(echo $INSPECT | jq -r '.HostConfig.NetworkMode')

  # Check for violations
  if [ "$PRIVILEGED" = "true" ]; then
    echo "VIOLATION: $NAME — Running in PRIVILEGED mode"
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  if [ "$PID_MODE" = "host" ]; then
    echo "VIOLATION: $NAME — Running with HOST PID namespace"
    VIOLATIONS=$((VIOLATIONS + 1))
  fi

  if [ "$NET_MODE" = "host" ]; then
    echo "WARNING: $NAME — Running with HOST network mode"
  fi
done

echo ""
echo "Audit complete."
```

## Step 7: Portainer Security Policy Template

Document your policy as code:

```yaml
# portainer-security-policy.yml
# Security policy for all Portainer environments

policy_version: "1.0"
environments:
  production:
    bind_mounts: disabled
    privileged_mode: disabled
    host_pid: disabled
    host_network: disabled
    device_mapping: disabled
    sysctl_settings: disabled
    approved_registries:
      - registry.company.com
      - ghcr.io/your-org
    image_signing: required

  staging:
    bind_mounts: disabled
    privileged_mode: disabled
    host_pid: disabled
    host_network: disabled
    approved_registries:
      - registry.company.com
      - ghcr.io/your-org

  development:
    bind_mounts: allowed  # More permissive for dev
    privileged_mode: disabled
    host_pid: disabled
    approved_registries: any
```

## Conclusion

Docker security policies in Portainer provide a centralized enforcement mechanism for container security standards. Enable all applicable restrictions for production environments, run periodic security audits to detect violations, and use pre-approved application templates to guide developers toward secure defaults. Combine Portainer's security policies with Docker daemon hardening and registry controls for defense in depth.
