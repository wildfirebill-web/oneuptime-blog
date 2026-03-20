# How to Use the Edge Environment Waiting Room in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Waiting Room, Onboarding, Security

Description: Manage the Portainer Edge Waiting Room to review and approve new edge devices before they become active managed environments.

## Introduction

The Waiting Room in Portainer is a security gate for edge auto-onboarding. When new edge devices connect with a pre-staged edge key, they enter the waiting room instead of immediately becoming active environments. An administrator reviews and approves each device, preventing unauthorized devices from joining your management plane.

## Accessing the Waiting Room

1. Go to **Environments** in Portainer's left menu
2. Click the **Waiting Room** tab (or button)
3. Review pending devices

## What Appears in the Waiting Room

For each pending device:
- **Edge ID**: The device's identifier
- **IP Address**: The device's connecting IP
- **First Seen**: When the device first connected
- **Last Check-in**: Most recent check-in time
- **Tags**: Any tags pre-configured in the edge key

## Approving Devices via UI

1. Check the checkbox next to one or more devices
2. Click **Approve** to activate them as managed environments
3. The environment appears in the Environments list
4. Configure access policies after approval

## Bulk Operations via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# List waiting room devices

curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/edge/waiting-room \
  | python3 -c "
import sys, json
devices = json.load(sys.stdin)
print(f'Waiting devices: {len(devices)}')
for d in devices:
    print(f'  ID: {d[\"EndpointID\"]} | Name: {d.get(\"Name\",\"unnamed\")} | IP: {d.get(\"LastCheckInIP\",\"unknown\")}')
"

# Approve specific device
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge/waiting-room/approve \
  -d '[1, 2, 3]'   # Array of EndpointIDs to approve

# Approve all waiting devices (use carefully)
WAITING_IDS=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/edge/waiting-room \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps([e['EndpointID'] for e in d]))")

curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/edge/waiting-room/approve \
  -d "$WAITING_IDS"

# Delete (reject) a device from waiting room
ENDPOINT_ID=5
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}"
```

## Automated Approval Based on Device Attributes

For trusted device fleets, approve programmatically based on criteria:

```bash
#!/bin/bash
# auto-approve-trusted-devices.sh

TOKEN="your-admin-token"
PORTAINER_URL="https://portainer.example.com"

# Get all waiting devices
WAITING=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/edge/waiting-room")

# Filter and approve devices with known IP prefix (e.g., internal network)
APPROVED_IDS=$(echo $WAITING | python3 -c "
import sys, json
devices = json.load(sys.stdin)
approved = [
    d['EndpointID'] for d in devices
    if d.get('LastCheckInIP', '').startswith('192.168.100.')  # Trusted network
]
print(json.dumps(approved))
")

if [ "$APPROVED_IDS" != "[]" ]; then
  echo "Approving devices: $APPROVED_IDS"
  curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    "${PORTAINER_URL}/api/edge/waiting-room/approve" \
    -d "$APPROVED_IDS"
fi
```

## Skipping the Waiting Room

For fully automated, trusted deployments, skip the waiting room entirely:

1. Settings → Edge Compute → Enable **Auto-approve edge devices**

This immediately activates all new devices without waiting room review.

## Conclusion

The Waiting Room provides a human review gate between device provisioning and management access. It's the right balance for most organizations: automated provisioning (devices self-register) with human oversight (admin approves). For fully automated deployment pipelines, the API-based approval and skip-waiting-room options provide the necessary flexibility.
