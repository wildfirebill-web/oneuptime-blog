# How to Use a Non-Default Admin Username in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Authentication, Hardening, DevOps

Description: Learn how to configure Portainer with a non-default admin username to reduce exposure to credential stuffing and brute force attacks targeting the default admin account.

## Introduction

Using `admin` as your Portainer administrator username is a well-known default that attackers specifically target in brute force and credential stuffing attacks. Changing to a unique, organization-specific username adds a layer of security through obscurity as part of a defense-in-depth strategy.

## Prerequisites

- Fresh Portainer installation (for initial setup)
- OR existing Portainer with admin access (for changing existing admin username)

## Method 1: Set Non-Default Username During Initial Setup

The best time to change the admin username is during the initial Portainer setup:

### Via the Web UI

1. Open Portainer for the first time at `https://portainer.example.com:9443`.
2. On the **Create initial administrator** page, change the **Username** field from `admin` to your chosen username.
3. Enter a strong password.
4. Click **Create user**.

### Via the API (for automated setups)

```bash
PORTAINER_URL="https://portainer.example.com:9443"
ADMIN_USERNAME="portainer-ops"  # Custom, non-default username
ADMIN_PASSWORD="$(openssl rand -base64 32)"

# Create the initial admin with custom username
curl -s -X POST "${PORTAINER_URL}/api/users/admin/init" \
  -H "Content-Type: application/json" \
  -d "{
    \"username\": \"${ADMIN_USERNAME}\",
    \"password\": \"${ADMIN_PASSWORD}\"
  }" | jq .

echo "Admin created: ${ADMIN_USERNAME}"
echo "Password: ${ADMIN_PASSWORD}"
# Store the password securely!
```

## Method 2: Add a New Admin and Remove the Default

If Portainer is already set up with the default `admin` user:

### Step 1: Create a New Admin User

```bash
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"currentpassword"}' | jq -r '.jwt')

# Create a new admin account
NEW_ADMIN=$(curl -s -X POST https://portainer.example.com/api/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "portainer-ops",
    "password": "NewSecurePassword123!",
    "role": 1
  }')

NEW_ADMIN_ID=$(echo $NEW_ADMIN | jq -r '.Id')
echo "New admin created with ID: $NEW_ADMIN_ID"
```

### Step 2: Verify New Admin Login Works

```bash
# Test authentication with the new admin
NEW_TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"portainer-ops","password":"NewSecurePassword123!"}' | jq -r '.jwt')

if [ -n "$NEW_TOKEN" ] && [ "$NEW_TOKEN" != "null" ]; then
  echo "New admin authentication successful!"
else
  echo "ERROR: New admin login failed. Do NOT proceed with deleting old admin."
  exit 1
fi
```

### Step 3: Delete the Default Admin Account

```bash
# Use the new admin token to delete the old admin
# First, find the old admin's user ID
OLD_ADMIN_ID=$(curl -s -H "Authorization: Bearer $NEW_TOKEN" \
  https://portainer.example.com/api/users | \
  jq -r '.[] | select(.Username == "admin") | .Id')

echo "Deleting old admin (ID: $OLD_ADMIN_ID)..."

curl -s -X DELETE -H "Authorization: Bearer $NEW_TOKEN" \
  "https://portainer.example.com/api/users/${OLD_ADMIN_ID}"

echo "Default admin account deleted."
```

## Method 3: Rename Existing Admin (Limited Support)

Portainer doesn't directly support renaming users. The create-new + delete-old pattern is the standard approach.

## Choosing a Good Admin Username

Avoid predictable usernames:
- `admin`, `administrator`, `root`, `superuser` — Targeted by default
- Your company name alone — Too predictable
- `portainer` — Known default for some setups

Use less predictable options:
- `infra-ops` — Function-based
- `platform-admin` — Team-based
- `sre-lead` — Role-based with initials
- A random identifier: `portal-a7k2m` — Hardest to guess

## Scripted Initial Setup with Custom Username

```bash
#!/bin/bash
# portainer-init-secure.sh — Initialize with custom admin username

PORTAINER_URL="${PORTAINER_URL:-https://portainer.example.com}"

# Generate secure random credentials
ADMIN_USERNAME="${ADMIN_USERNAME:-portainer-ops-$(openssl rand -hex 4)}"
ADMIN_PASSWORD="${ADMIN_PASSWORD:-$(openssl rand -base64 24)}"

echo "Waiting for Portainer to start..."
until curl -sf "${PORTAINER_URL}/api/system/status" > /dev/null 2>&1; do
  sleep 3
done

echo "Creating admin user: ${ADMIN_USERNAME}"

RESPONSE=$(curl -s -X POST "${PORTAINER_URL}/api/users/admin/init" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${ADMIN_USERNAME}\",\"password\":\"${ADMIN_PASSWORD}\"}")

if echo "$RESPONSE" | jq -e '.Id' > /dev/null 2>&1; then
  echo "Admin created successfully!"
  echo ""
  echo "=== Credentials (store these securely) ==="
  echo "URL:      ${PORTAINER_URL}"
  echo "Username: ${ADMIN_USERNAME}"
  echo "Password: ${ADMIN_PASSWORD}"

  # Save to a secrets file (ensure this file is secured)
  echo "PORTAINER_ADMIN_USER=${ADMIN_USERNAME}" > /run/secrets/portainer-admin
  echo "PORTAINER_ADMIN_PASS=${ADMIN_PASSWORD}" >> /run/secrets/portainer-admin
  chmod 600 /run/secrets/portainer-admin
else
  echo "ERROR: ${RESPONSE}"
  exit 1
fi
```

## Conclusion

Using a non-default admin username in Portainer is a simple but effective security measure that reduces your exposure to automated attacks targeting default credentials. Set the custom username during initial setup for the cleanest approach, or use the create-new + delete-old pattern for existing installations. Combine this with a strong password, HTTPS, IP restrictions, and RBAC for a comprehensive security posture.
