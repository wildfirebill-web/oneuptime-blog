# How to Generate a Support Bundle in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, Support, Troubleshooting, Diagnostic

Description: Learn how to generate a Portainer Business Edition support bundle containing diagnostic information for escalating issues to Portainer support.

---

When troubleshooting complex issues in Portainer BE, the support team may request a support bundle - a compressed archive containing logs, configuration details, and diagnostic data that helps identify the root cause without requiring direct access to your system.

## What's in a Support Bundle

A Portainer support bundle typically includes:
- Portainer application logs
- Database diagnostic information
- Environment configuration (sanitized)
- Version and license information
- System resource information

Sensitive data (passwords, secrets) is excluded or redacted.

## Generate a Support Bundle via the UI

### Method 1: Settings Menu (BE)

1. Log in as an administrator
2. Navigate to **Settings**
3. Scroll to **Support** or **Diagnostics**
4. Click **Generate Support Bundle**
5. Download the resulting `.zip` or `.tar.gz` file

### Method 2: Help Menu

Some Portainer BE versions include a support bundle option in the **?** (Help) menu in the top navigation bar.

## Generate via API

```bash
# Authenticate

TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Generate and download support bundle
curl -X GET \
  https://localhost:9443/api/system/supportbundle \
  -H "Authorization: Bearer $TOKEN" \
  --output portainer_support_bundle_$(date +%Y%m%d_%H%M%S).zip \
  --insecure

echo "Support bundle downloaded"
ls -lh portainer_support_bundle_*.zip
```

## Manual Log Collection

If the support bundle feature is unavailable, manually collect the required information:

```bash
#!/bin/bash
# collect-portainer-diagnostics.sh

OUTPUT_DIR="portainer_diagnostics_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "Collecting Portainer diagnostics..."

# Container logs (last 1000 lines)
docker logs --tail 1000 portainer > "$OUTPUT_DIR/portainer.log" 2>&1

# Container inspect
docker inspect portainer > "$OUTPUT_DIR/container_inspect.json"

# Docker info
docker info > "$OUTPUT_DIR/docker_info.txt" 2>&1

# Container stats snapshot
docker stats portainer --no-stream > "$OUTPUT_DIR/container_stats.txt"

# Portainer version via API
curl -sk https://localhost:9443/api/status > "$OUTPUT_DIR/portainer_status.json"

# System info
uname -a > "$OUTPUT_DIR/system_info.txt"
free -h >> "$OUTPUT_DIR/system_info.txt"
df -h >> "$OUTPUT_DIR/system_info.txt"

# Compress everything
tar czf "${OUTPUT_DIR}.tar.gz" "$OUTPUT_DIR"
rm -rf "$OUTPUT_DIR"

echo "Diagnostics bundle: ${OUTPUT_DIR}.tar.gz"
ls -lh "${OUTPUT_DIR}.tar.gz"
```

## Sharing the Support Bundle

When sending the bundle to Portainer support:
- Open a ticket at support.portainer.io
- Attach the bundle file (or provide a download link)
- Include a description of the issue and steps to reproduce
- Note your Portainer version and deployment type

---

*Proactively monitor your Portainer infrastructure with [OneUptime](https://oneuptime.com) before issues require support escalation.*
