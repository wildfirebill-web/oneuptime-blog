# How to Configure Casdoor SSO with Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Casdoor, SSO, OAuth, Self-Hosted, Authentication

Description: Set up Casdoor as an OIDC provider for Portainer, enabling self-hosted SSO with Casdoor's multi-provider identity management.

## Introduction

Casdoor is an open-source centralized authentication/authorization platform with a web-based admin UI. It supports OIDC, OAuth 2.0, and SAML, and can federate with dozens of upstream providers. This guide configures Casdoor as Portainer's OIDC provider.

## Prerequisites

- Casdoor deployed and accessible
- Portainer running on HTTPS
- Casdoor admin access

## Step 1: Deploy Casdoor

Quick deployment with Docker:

```yaml
# casdoor-stack.yml

version: "3.8"

services:
  casdoor:
    image: casbin/casdoor:latest
    container_name: casdoor
    environment:
      RUNNING_IN_DOCKER: "true"
    ports:
      - "8000:8000"
    volumes:
      - casdoor_data:/casdoor/conf
    depends_on:
      - casdoor-db

  casdoor-db:
    image: mysql:8.0
    container_name: casdoor-db
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: casdoor
    volumes:
      - casdoor_db_data:/var/lib/mysql

volumes:
  casdoor_data:
  casdoor_db_data:
```

```bash
docker compose -f casdoor-stack.yml up -d
# Access at http://localhost:8000 (admin/123)
```

## Step 2: Create an Application in Casdoor

1. Log in to Casdoor admin UI (`http://localhost:8000`)
2. Go to **Applications** → **Add**

Fill in:
```sql
Name:          portainer
DisplayName:   Portainer
Organization:  built-in
Logo:          https://portainer.io/images/logos/portainer-icon.png

Redirect URLs:
  https://portainer.example.com/

Grant Types:
  ✓ Authorization Code

Providers:
  (select your configured providers - local, LDAP, etc.)
```

3. Save and note the **Client ID** and **Client Secret**

## Step 3: Get Casdoor OIDC Endpoints

```bash
CASDOOR_URL="https://casdoor.example.com"

# Discovery endpoint
curl -s "${CASDOOR_URL}/.well-known/openid-configuration" | python3 -m json.tool

# Casdoor endpoints:
# Authorization: ${CASDOOR_URL}/login/oauth/authorize
# Token:         ${CASDOOR_URL}/api/login/oauth/access_token
# UserInfo:      ${CASDOOR_URL}/api/userinfo
```

## Step 4: Configure Portainer for Casdoor

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

CASDOOR_URL="https://casdoor.example.com"
CLIENT_ID="your-casdoor-client-id"
CLIENT_SECRET="your-casdoor-client-secret"

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d "{
    \"AuthenticationMethod\": 3,
    \"oauthsettings\": {
      \"ClientID\": \"${CLIENT_ID}\",
      \"ClientSecret\": \"${CLIENT_SECRET}\",
      \"AuthorizationURI\": \"${CASDOOR_URL}/login/oauth/authorize\",
      \"AccessTokenURI\": \"${CASDOOR_URL}/api/login/oauth/access_token\",
      \"ResourceURI\": \"${CASDOOR_URL}/api/userinfo\",
      \"RedirectURI\": \"https://portainer.example.com/\",
      \"UserIdentifier\": \"name\",
      \"Scopes\": \"openid profile email\",
      \"OAuthAutoCreateUsers\": true,
      \"SSO\": true
    }
  }"
```

Note: Casdoor uses `name` as the username identifier in userinfo responses.

## Step 5: Add Users to Casdoor

1. In Casdoor admin → **Users** → **Add**
2. Or sync from LDAP/AD via Casdoor's user provider

```bash
# Casdoor API - Create a user
curl -X POST \
  -H "Content-Type: application/json" \
  "${CASDOOR_URL}/api/add-user" \
  -d '{
    "owner": "built-in",
    "name": "alice",
    "displayName": "Alice Smith",
    "email": "alice@example.com",
    "password": "AlicePassword123",
    "type": "normal-user"
  }'
```

## Step 6: Configure Casdoor with LDAP (Optional)

Connect Casdoor to LDAP for corporate SSO:

1. In Casdoor → **Providers** → **Add**
2. Choose **LDAP** as provider type
3. Configure LDAP settings
4. Link the LDAP provider to the Portainer application

This allows users to log in to Casdoor (and thus Portainer) with LDAP credentials.

## Testing the Integration

1. Log out of Portainer
2. Click the OAuth login button
3. Casdoor's login page appears
4. Log in with a Casdoor user
5. Should be redirected back to Portainer

```bash
# Verify Casdoor's userinfo endpoint returns expected claims
CASDOOR_ACCESS_TOKEN="your-test-token"
curl -H "Authorization: Bearer $CASDOOR_ACCESS_TOKEN" \
  "${CASDOOR_URL}/api/userinfo" | python3 -m json.tool
```

## Conclusion

Casdoor provides a versatile self-hosted SSO solution that can aggregate multiple identity sources (LDAP, social logins, etc.) and present a unified OIDC interface to Portainer. Its clean admin UI makes application and user management straightforward. When combined with Casdoor's LDAP federation, you get enterprise SSO without needing Keycloak's complexity.
