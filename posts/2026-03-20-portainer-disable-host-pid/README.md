# How to Disable Host PID Access for Non-Admin Users in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Docker, Hardening, Container

Description: Learn how to prevent non-admin users from running containers with host PID namespace sharing in Portainer, blocking a critical container escape vector.

## Introduction

The `--pid=host` option in Docker shares the host's process namespace with a container. This allows a container to see and interact with all processes on the host, including sensitive system processes. In a shared Portainer environment, this is a critical security risk. Portainer allows administrators to disable this for non-admin users.

## Why Host PID Is Dangerous

When a container runs with `--pid=host`:

```bash
# Inside a container with --pid=host, the user can:
# See all host processes
ps aux  # Shows ALL host processes

# Kill host processes
kill -9 HOST_PID  # Can crash the host OS or services

# Attach to host processes (with ptrace capability)
strace -p HOST_PROCESS_PID

# Access /proc/[pid]/environ of host processes — may expose secrets
cat /proc/1/environ  # Read init process environment variables

# Read memory of other processes
cat /proc/HOST_PID/mem  # With appropriate permissions
```

## Step 1: Disable Host PID in Portainer

### Via Portainer UI

1. Log into Portainer as admin.
2. Select your Docker environment.
3. Go to environment **Settings** (gear icon).
4. Scroll to the **Security** section.
5. Find the **Disable host PID namespace for non-administrators** toggle.
6. Enable it.
7. Click **Save environment settings**.

### Via Portainer API

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-admin-token"
ENDPOINT_ID=1

# Disable host PID for non-admin users
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/settings" \
  -d '{
    "disableHostPidForRegularUsers": true
  }' | jq .

echo "Host PID access disabled for non-admin users on endpoint $ENDPOINT_ID"
```

## Step 2: Understand What Gets Blocked

After enabling this restriction:

- Non-admin users **cannot** create containers with `--pid=host`
- Non-admin users **cannot** create containers with `"PidMode": "host"` via Docker Compose
- Admin users retain the ability to use host PID when genuinely required

Legitimate use cases for host PID (admin-only):

```bash
# Performance profiling tools that need host PID visibility
docker run --pid=host --name profiler nicholasgasior/nstat

# System monitoring containers (node exporters, etc.)
docker run --pid=host -v /proc:/host/proc:ro prom/node-exporter

# Docker-in-Docker scenarios (with appropriate controls)
```

## Step 3: Apply Across All Environments

```bash
#!/bin/bash
# apply-security-restrictions.sh — Apply all restrictions to all environments

PORTAINER_URL="https://portainer.example.com"
TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# Get all Docker environments (Type 1 or 2)
ENDPOINTS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints" | \
  jq -c '.[] | select(.Type == 1 or .Type == 2)')

echo "$ENDPOINTS" | while read -r ENDPOINT; do
  ID=$(echo $ENDPOINT | jq -r '.Id')
  NAME=$(echo $ENDPOINT | jq -r '.Name')

  RESULT=$(curl -s -X PUT \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/endpoints/${ID}/settings" \
    -d '{
      "disableBindMountsForRegularUsers": true,
      "disablePrivilegeModeForRegularUsers": true,
      "disableHostPidForRegularUsers": true,
      "disableHostNetworkForRegularUsers": true
    }')

  echo "Applied security restrictions to: $NAME (ID: $ID)"
done

echo "All environments secured."
```

## Step 4: Comprehensive Security Restrictions

Disable host PID as part of a broader security hardening:

```bash
# Apply all container security restrictions via API
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
    "disableSysctlSettingForRegularUsers": true
  }'
```

## Step 5: Docker Security Defaults

Teach users to rely on secure container defaults:

```yaml
# Secure docker-compose.yml — No host namespace sharing needed
version: "3.8"
services:
  app:
    image: myapp:latest
    restart: unless-stopped
    # No pid: host
    # No network_mode: host
    # No privileged: true
    security_opt:
      - no-new-privileges:true  # Prevent privilege escalation
    cap_drop:
      - ALL                     # Drop all capabilities
    cap_add:
      - NET_BIND_SERVICE        # Add only what's needed
    read_only: true             # Read-only root filesystem
    tmpfs:
      - /tmp                    # Writable temp storage
    ports:
      - "3000:3000"             # Expose via port mapping, not host network
```

## Step 6: Monitor for Violation Attempts

Check Portainer logs for attempts to create containers with restricted options:

```bash
# Monitor Portainer logs for security-related events
docker logs portainer 2>&1 | grep -i "pid\|privilege\|bind\|restricted" | tail -20

# Or in a loop for real-time monitoring
docker logs -f portainer 2>&1 | grep -i "restricted\|denied\|unauthorized"
```

## Conclusion

Disabling host PID access for non-admin users in Portainer eliminates a significant container escape vector. Combined with disabling privileged mode, bind mounts, and host network mode, these restrictions create a strong multi-layered container security policy. Admin users retain full flexibility for legitimate operational tools, while standard users operate within a safe, isolated container environment.
