# How to Configure Session Timeout Duration in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Session Management, Administration, Configuration

Description: Set the user session timeout in Portainer to automatically log out inactive users and meet security compliance requirements.

## Introduction

Session timeout controls how long a user can remain logged into Portainer without activity before being automatically signed out. This is a critical security control for shared workstations, compliance requirements (PCI-DSS, HIPAA, SOC2), and general security hygiene. Portainer allows configuring this via both the UI and CLI flags.

## Default Session Timeout

Portainer's default session timeout is **8 hours**. For production environments, this may be too long.

## Method 1: CLI Flag (Applies to All Users)

Set the session timeout at startup. This applies globally to all users:

```bash
# Set session timeout to 1 hour

docker run -d \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --http-enabled \
  --trusted-origins=https://portainer.example.com \
  --http-disabled-password \
  --feature-flag-user-timeout=1h
```

Common duration format strings:

```text
1h      = 1 hour
30m     = 30 minutes
2h30m   = 2.5 hours
24h     = 24 hours (1 day)
168h    = 1 week
```

## Method 2: Settings UI (Recommended)

1. Log in as an administrator
2. Go to **Settings** → **Authentication**
3. Find **User Session Timeout**
4. Choose from preset options or enter a custom duration:
   - `1h` - 1 hour
   - `4h` - 4 hours (good default for most teams)
   - `8h` - 8 hours (Portainer default)
   - `24h` - 24 hours
   - `168h` - 1 week

## Method 3: API Configuration

```bash
# Authenticate
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Update session timeout to 2 hours
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "UserSessionTimeout": "2h"
  }'

# Verify the change
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/settings \
  | python3 -c "import sys,json; s=json.load(sys.stdin); print('Timeout:', s.get('UserSessionTimeout','not set'))"
```

## Docker Compose Configuration

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    command:
      - "--http-enabled"
      - "--trusted-origins=https://portainer.example.com"
    networks:
      - proxy

volumes:
  portainer_data:
```

Then configure via the Settings UI after deployment for easier management.

## Session Timeout vs. Token Expiry

It's important to understand the difference:

| Setting | What It Controls |
|---------|-----------------|
| **Session Timeout** | How long the UI session lasts after the last user action |
| **JWT Token Expiry** | How long the API token remains valid (tied to session timeout) |
| **API Access Tokens** | Personal access tokens do not expire based on session timeout |

API access tokens (generated in **Account** → **Access tokens**) are not affected by session timeout - they have their own expiry if set.

## Compliance Recommendations

**HIPAA / Healthcare**: Maximum 15 minutes inactivity timeout is common.

**PCI-DSS**: Requires session termination after no more than 15 minutes of inactivity.

**NIST 800-53**: Recommends automatic session termination after defined inactivity.

```bash
# PCI-DSS compliant setting (15 minutes)
--UserSessionTimeout=15m
```

## Monitoring Session Activity

Track session activity via Portainer audit logs (Business Edition):

```bash
# Get recent authentication events via API
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/audit?limit=20&action=login" \
  | python3 -m json.tool
```

## Conclusion

Configuring session timeout is a simple but important security measure. Start with the Settings UI for the most straightforward approach, and use the API for automation. For compliance-driven environments, ensure the timeout is set to meet your specific framework's requirements, and document the setting as part of your security controls.
