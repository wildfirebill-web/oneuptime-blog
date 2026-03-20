# How to Add a Login Screen Banner in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, customization, security, banner, login, compliance

Description: A guide to adding a login screen banner in Portainer for compliance requirements, security notices, or informational messages.

## Overview

Portainer Business Edition allows you to display a custom message on the login screen. This is commonly used for legal notices ("Authorized users only"), security warnings, maintenance announcements, or organizational information. Login banners are often required by compliance frameworks (SOX, HIPAA, government systems) to inform users about acceptable use policies.

## Prerequisites

- Portainer Business Edition
- Admin access to Portainer

## Step 1: Configure Login Screen Banner via UI

1. Log in to Portainer as admin
2. Navigate to **Settings** → **Appearance**
3. Find the **Login Screen** section
4. Toggle **Enable login screen message**
5. Enter your message in the text area
6. Click **Save**

The message supports basic text formatting and will appear below the login form.

## Step 2: Configure via Portainer API

```bash
PORTAINER_URL="https://portainer.example.com:9443"
TOKEN="your-admin-token"

# Set login screen message
curl -X PUT \
  "${PORTAINER_URL}/api/settings" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "ShowKomposeBuildOption": false,
    "customLoginBanner": "AUTHORIZED USE ONLY\n\nThis system is for authorized users only. By logging in, you agree to comply with organizational security policies. Unauthorized access is prohibited and may be subject to legal action."
  }'
```

## Common Banner Templates

### Security Notice (Government/Compliance)

```
NOTICE: This system is for authorized users only.

Individuals using this computer system without authority, or in excess of their authority, are subject to monitoring and recording. Anyone using this system expressly consents to such monitoring.

Evidence of unauthorized use may be provided to law enforcement.
```

### Maintenance Notice

```
SCHEDULED MAINTENANCE: This system will be unavailable on Saturday, March 22, 2026 from 02:00-04:00 UTC for maintenance.

For urgent support, contact: ops@example.com
```

### HIPAA Compliance Notice

```
HIPAA NOTICE: This system contains Protected Health Information (PHI).

Access is restricted to authorized personnel only. All access is logged and audited. Unauthorized access or disclosure of PHI is a violation of HIPAA regulations and may result in civil and criminal penalties.

By logging in, you acknowledge that you have received HIPAA training and agree to handle PHI in accordance with all applicable regulations.
```

### PCI DSS Notice

```
RESTRICTED SYSTEM: This system processes payment card data.

Access is granted to authorized personnel only. All activities are monitored and logged. This system must be used in compliance with PCI DSS requirements.

Unauthorized access is strictly prohibited.
```

## Step 3: View Current Banner Settings

```bash
# Retrieve current settings
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${PORTAINER_URL}/api/settings" \
  | jq '.customLoginBanner'
```

## Step 4: Clear/Remove Login Banner

```bash
# Via UI: Settings → Appearance → Disable login screen message

# Via API
curl -X PUT \
  "${PORTAINER_URL}/api/settings" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"customLoginBanner": ""}'
```

## Automating Banner Updates

```bash
#!/bin/bash
# Update banner during maintenance windows
PORTAINER_URL="https://portainer.example.com:9443"
TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"'"${PORTAINER_PASS}"'"}' \
  | jq -r '.jwt')

# Set maintenance banner
curl -X PUT "${PORTAINER_URL}/api/settings" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"customLoginBanner\": \"MAINTENANCE IN PROGRESS: Expected completion $(date -d '+2 hours' '+%H:%M %Z')\"}"
```

## Conclusion

Login screen banners are a simple but important security control that signals to users that the system is monitored and access is restricted. Portainer Business Edition makes it easy to configure banners via the UI or API. For compliance-driven organizations, ensure your banner text has been reviewed by your legal and security teams to meet specific regulatory requirements.
