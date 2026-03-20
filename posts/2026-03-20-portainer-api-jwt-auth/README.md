# How to Authenticate with the Portainer API Using JWT Tokens

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Authentication, JWT, Security

Description: Learn how to authenticate with the Portainer REST API using JSON Web Tokens (JWT), including token acquisition, usage, expiry handling, and security best practices.

## Introduction

The Portainer API uses JWT (JSON Web Token) based authentication as its primary authentication method. Every API request (except the initial login) must include a valid JWT in the `Authorization` header. This guide covers how to obtain a JWT, use it in requests, handle expiry, and follow security best practices.

## Prerequisites

- Portainer CE or BE running and accessible
- Valid Portainer admin or user credentials
- `curl` and `jq` installed on your machine

## Step 1: Obtain a JWT Token

Send a POST request to the `/api/auth` endpoint with your credentials:

```bash
# Basic authentication - get JWT token
RESPONSE=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "yourpassword"
  }')

echo "Full response: $RESPONSE"

# Extract just the JWT token
TOKEN=$(echo $RESPONSE | jq -r '.jwt')
echo "JWT Token: $TOKEN"
```

A successful response looks like:

```json
{
  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6MSwiZXhwIjoxNzExNjQ5NjAwLCJpYXQiOjE3MTE2NDYwMDB9.SIGNATURE"
}
```

## Step 2: Include the JWT in API Requests

Include the token in the `Authorization` header as a `Bearer` token:

```bash
# Set the token as a variable for reuse
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Use in subsequent requests
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints | jq .

# List users
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/users | jq .

# Get system status
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/system/status | jq .
```

## Step 3: Inspect the JWT Token

A JWT contains three base64-encoded parts: header, payload, and signature:

```bash
# Decode the JWT payload to see its contents
decode_jwt() {
  local TOKEN=$1
  # Extract the payload (second segment)
  echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq .
}

decode_jwt $TOKEN

# Example decoded payload:
# {
#   "username": "admin",
#   "role": 1,           # 1 = admin, 2 = standard user
#   "iat": 1711646000,   # Issued at timestamp
#   "exp": 1711649600    # Expiry timestamp
# }
```

## Step 4: Check Token Expiry

```bash
# Check when your token expires
check_token_expiry() {
  local TOKEN=$1
  EXP=$(echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq -r '.exp')
  NOW=$(date +%s)
  REMAINING=$((EXP - NOW))

  if [ $REMAINING -le 0 ]; then
    echo "Token has EXPIRED"
  else
    echo "Token expires in ${REMAINING} seconds ($(date -d @$EXP 2>/dev/null || date -r $EXP))"
  fi
}

check_token_expiry $TOKEN
```

## Step 5: Handle Token Expiry in Scripts

JWT tokens from Portainer expire after a configurable period (default: 8 hours). Build token refresh logic into your scripts:

```bash
#!/bin/bash
# portainer-helper.sh — Reusable authentication with auto-refresh

PORTAINER_URL="https://portainer.example.com"
PORTAINER_USER="${PORTAINER_USER:-admin}"
PORTAINER_PASS="${PORTAINER_PASS:-password}"
TOKEN_FILE="/tmp/portainer-token.json"

get_token() {
  RESPONSE=$(curl -s -X POST "${PORTAINER_URL}/api/auth" \
    -H "Content-Type: application/json" \
    -d "{\"username\":\"${PORTAINER_USER}\",\"password\":\"${PORTAINER_PASS}\"}")

  TOKEN=$(echo $RESPONSE | jq -r '.jwt')
  EXP=$(echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq -r '.exp')

  echo "{\"token\": \"$TOKEN\", \"exp\": $EXP}" > "$TOKEN_FILE"
  echo $TOKEN
}

get_valid_token() {
  # Check if we have a cached, non-expired token
  if [ -f "$TOKEN_FILE" ]; then
    TOKEN=$(cat "$TOKEN_FILE" | jq -r '.token')
    EXP=$(cat "$TOKEN_FILE" | jq -r '.exp')
    NOW=$(date +%s)
    BUFFER=300  # Refresh 5 minutes before expiry

    if [ $((EXP - NOW)) -gt $BUFFER ]; then
      echo $TOKEN
      return
    fi
  fi

  # Token missing or expired — get a new one
  get_token
}

# Usage
TOKEN=$(get_valid_token)

curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints" | jq .
```

## Step 6: Using JWT in Different Tools

### Python

```python
import requests

def get_portainer_token(url, username, password):
    """Authenticate with Portainer and return JWT token."""
    response = requests.post(
        f"{url}/api/auth",
        json={"username": username, "password": password},
        verify=True  # Set to False only for self-signed certs in dev
    )
    response.raise_for_status()
    return response.json()["jwt"]

# Usage
url = "https://portainer.example.com"
token = get_portainer_token(url, "admin", "yourpassword")

headers = {"Authorization": f"Bearer {token}"}

# Make authenticated requests
endpoints = requests.get(f"{url}/api/endpoints", headers=headers)
print(endpoints.json())
```

### JavaScript/Node.js

```javascript
const axios = require('axios');

async function getPortainerToken(url, username, password) {
  const response = await axios.post(`${url}/api/auth`, {
    username,
    password
  });
  return response.data.jwt;
}

// Usage
(async () => {
  const token = await getPortainerToken(
    'https://portainer.example.com',
    'admin',
    'yourpassword'
  );

  const headers = { Authorization: `Bearer ${token}` };

  // Make authenticated requests
  const { data } = await axios.get(
    'https://portainer.example.com/api/endpoints',
    { headers }
  );
  console.log(data);
})();
```

## Security Best Practices

1. **Never hardcode credentials** in scripts — use environment variables or a secrets manager
2. **Use HTTPS** for all API calls — never send credentials over HTTP
3. **Store tokens securely** — in memory or temp files with restricted permissions
4. **Set short token expiry** in Portainer admin settings for sensitive environments
5. **Use API access tokens** for CI/CD pipelines instead of username/password JWT auth

```bash
# Secure credential handling
export PORTAINER_USER="admin"
export PORTAINER_PASS="$(cat /run/secrets/portainer_password)"

TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${PORTAINER_USER}\",\"password\":\"${PORTAINER_PASS}\"}" | jq -r '.jwt')
```

## Conclusion

JWT authentication is the standard way to interact with the Portainer API. Obtain a token via the `/api/auth` endpoint, include it in every request's `Authorization: Bearer` header, and implement auto-refresh logic for long-running scripts. For automated pipelines, consider using Portainer's API access tokens which don't expire automatically and don't require credential storage.
