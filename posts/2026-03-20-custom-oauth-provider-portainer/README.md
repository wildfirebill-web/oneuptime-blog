# How to Set Up a Custom OAuth Provider with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Custom Provider, OpenID Connect, Authentication

Description: Configure Portainer to use any OAuth 2.0 or OpenID Connect provider by manually specifying the authorization, token, and userinfo endpoints.

---

Portainer's custom OAuth configuration supports any standards-compliant OAuth 2.0 or OpenID Connect provider. If your provider isn't listed as a preset, configure it manually using the provider's documented endpoints.

## Finding Your Provider's OAuth Endpoints

Most OpenID Connect providers publish their endpoints at a discovery URL:

```bash
# OpenID Connect providers expose a discovery document
# Replace with your provider's base URL
curl https://your-provider.example.com/.well-known/openid-configuration | \
  python3 -m json.tool | grep -E "authorization|token|userinfo"

# Example output:
# "authorization_endpoint": "https://your-provider.example.com/oauth2/authorize",
# "token_endpoint": "https://your-provider.example.com/oauth2/token",
# "userinfo_endpoint": "https://your-provider.example.com/oauth2/userinfo",
```

## Registering Portainer with Your Provider

In your OAuth provider, create a new application/client:
- **Type**: Web application (confidential client)
- **Redirect URI**: `https://portainer.example.com/`
- **Grant type**: Authorization Code
- **Scopes**: `openid profile email` (or provider-specific equivalents)

## Configure Custom OAuth in Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure custom OAuth provider
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 3,
    "OAuthSettings": {
      "ClientID": "portainer-client-id",
      "ClientSecret": "portainer-client-secret",
      "AuthorizationURI": "https://idp.example.com/oauth2/authorize",
      "AccessTokenURI": "https://idp.example.com/oauth2/token",
      "ResourceURI": "https://idp.example.com/oauth2/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "sub",
      "Scopes": "openid profile email",
      "OAuthAutoCreateUsers": true,
      "DefaultTeamID": 0
    }
  }' \
  --insecure
```

## User Identifier Selection

The `UserIdentifier` field tells Portainer which claim from the userinfo response to use as the username:

| Claim | Format | Use When |
|-------|--------|---------|
| `sub` | Opaque ID | Always unique, good for stable IDs |
| `email` | user@domain.com | Human-readable, may change |
| `preferred_username` | username | User-friendly, from OIDC |
| `login` | username | GitHub-style username |

Test your provider's userinfo response:

```bash
# Get an access token first, then check the userinfo endpoint
# Replace with actual access token after OAuth flow
ACCESS_TOKEN="eyJ..."

curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://idp.example.com/oauth2/userinfo | python3 -m json.tool
```

## Handling Non-Standard Providers

Some providers return user data at a different format or location:

```bash
# If the provider doesn't use standard userinfo endpoint,
# some providers embed user data in the access token (JWT)
# or return it in a non-standard response

# Test the access token endpoint to see what's returned
curl -X POST \
  https://idp.example.com/oauth2/token \
  -d "grant_type=authorization_code&code=AUTH_CODE&..." | \
  python3 -m json.tool
```

## Troubleshoot Custom OAuth

```bash
# Check Portainer logs for OAuth errors
docker logs portainer 2>&1 | grep -i "oauth\|token\|auth" | tail -30

# Verify the redirect URI matches exactly what's registered
# Common mistake: missing or extra trailing slash
```

---

*Extend your identity-managed Portainer with monitoring from [OneUptime](https://oneuptime.com).*
