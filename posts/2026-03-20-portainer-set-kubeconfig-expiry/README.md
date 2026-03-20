# How to Set Kubeconfig Expiry in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Security, kubeconfig, DevOps

Description: Learn how to configure kubeconfig token expiry in Portainer Business Edition to enforce periodic re-authentication and improve cluster security.

## Introduction

Kubeconfig files downloaded from Portainer contain access tokens that grant kubectl access to your Kubernetes clusters. By default, these tokens may not expire, which poses a security risk if a kubeconfig file is leaked or lost. Portainer Business Edition allows admins to set a maximum token lifetime, forcing users to periodically re-authenticate and download fresh kubeconfig files.

## Prerequisites

- Portainer Business Edition
- Admin access to Portainer
- A Kubernetes environment connected to Portainer

## Why Set Kubeconfig Expiry?

- **Security**: Limits the window of exposure if credentials are compromised
- **Compliance**: Many security frameworks require periodic credential rotation
- **Access revocation**: Expired tokens become invalid, ensuring departed team members lose access
- **Audit trail**: Forces re-authentication through Portainer, creating auth log entries

## Step 1: Navigate to Environment Settings

1. Log into Portainer as an administrator.
2. From the left sidebar, click **Environments**.
3. Find your Kubernetes environment and click on it to open its settings.
4. Alternatively, click the **gear icon** next to the environment in the home dashboard.

## Step 2: Configure Kubeconfig Expiry

1. In the environment settings, scroll to the **Security** section.
2. Find the **Kubeconfig expiry** field.
3. Choose a duration from the dropdown or enter a custom value:
   - `0` — No expiry (not recommended for production)
   - `4h` — Expires in 4 hours
   - `8h` — Expires in 8 hours
   - `24h` — Expires in 24 hours (common for daily use)
   - `7d` — Expires in 7 days
   - `30d` — Expires in 30 days
4. Click **Save environment settings**.

## Step 3: Verify the Expiry Is Applied

After setting expiry, new kubeconfigs downloaded by users will contain time-limited tokens. You can verify this by inspecting the downloaded kubeconfig:

```bash
# Download a fresh kubeconfig
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/config" \
  -o test-kubeconfig.yaml

# View the token field
cat test-kubeconfig.yaml | grep token:
```

The token is a JWT. Decode it to check its expiry:

```bash
# Decode the JWT to see expiry (exp claim)
TOKEN_VALUE=$(cat test-kubeconfig.yaml | grep 'token:' | awk '{print $2}')

# Decode the payload (base64 decode the middle segment)
echo $TOKEN_VALUE | cut -d'.' -f2 | base64 -d 2>/dev/null | jq '{iat: .iat, exp: .exp}'

# Convert exp to human-readable date
EXP=$(echo $TOKEN_VALUE | cut -d'.' -f2 | base64 -d 2>/dev/null | jq -r '.exp')
date -d @$EXP  # Linux
# date -r $EXP  # macOS
```

## Step 4: Handle Token Expiry in Workflows

When a token expires, kubectl commands return:

```
error: You must be logged in to the server (Unauthorized)
```

Users must re-download their kubeconfig from Portainer. Automate this with a refresh script:

```bash
#!/bin/bash
# auto-refresh-kubeconfig.sh

PORTAINER_URL="https://portainer.example.com"
PORTAINER_USER="${PORTAINER_USER:-myuser}"
PORTAINER_PASS="${PORTAINER_PASS:-mypassword}"
ENDPOINT_ID="${ENDPOINT_ID:-1}"
KUBECONFIG_PATH="${HOME}/.kube/portainer.yaml"

echo "[$(date)] Refreshing kubeconfig from Portainer..."

# Authenticate
TOKEN=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${PORTAINER_USER}\",\"password\":\"${PORTAINER_PASS}\"}" | jq -r '.jwt')

if [ -z "$TOKEN" ] || [ "$TOKEN" = "null" ]; then
  echo "ERROR: Failed to authenticate with Portainer" >&2
  exit 1
fi

# Download kubeconfig
HTTP_STATUS=$(curl -s -o "$KUBECONFIG_PATH" -w "%{http_code}" \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/kubernetes/config")

if [ "$HTTP_STATUS" = "200" ]; then
  echo "[$(date)] Kubeconfig refreshed successfully at $KUBECONFIG_PATH"
else
  echo "ERROR: Failed to download kubeconfig (HTTP $HTTP_STATUS)" >&2
  exit 1
fi
```

```bash
# Schedule the refresh before expiry
# If expiry is 24h, refresh every 20h to avoid interruption
# crontab -e
0 */20 * * * PORTAINER_USER=myuser PORTAINER_PASS=mypassword /opt/scripts/auto-refresh-kubeconfig.sh
```

## Best Practices for Kubeconfig Expiry

| Environment | Recommended Expiry |
|-------------|-------------------|
| Development | 7 days |
| Staging | 24 hours |
| Production | 8 hours |
| CI/CD pipelines | 1 hour (use API tokens instead) |

For CI/CD pipelines, prefer **Portainer API access tokens** over kubeconfig files, as they can be scoped more tightly and revoked independently.

## Revoking Access Immediately

If a user's access needs to be revoked before the token expires:

1. Go to **Users** in Portainer admin.
2. Find the user and click their name.
3. Either **delete the user** or **remove their environment access**.
4. All future API calls using their credentials will be rejected.

Note: Already-downloaded, unexpired kubeconfig files may still work until expiry. This is why short expiry windows are important.

## Conclusion

Setting kubeconfig expiry in Portainer is a simple but important security control. It limits the blast radius of credential exposure, enforces re-authentication through Portainer's audit trail, and helps meet compliance requirements. Combine short expiry windows with automated refresh scripts to maintain both security and developer productivity.
