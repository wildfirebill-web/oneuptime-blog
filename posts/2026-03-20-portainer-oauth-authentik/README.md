# How to Fix OAuth Login Issues with Authentik in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, OAuth, Authentik, SSO, Authentication, Troubleshooting

Description: Configure and troubleshoot OAuth 2.0 Single Sign-On between Portainer and Authentik identity provider, covering callback URL issues, scope configuration, and claim mapping.

## Introduction

Authentik is a popular self-hosted identity provider that many teams use for SSO. Integrating it with Portainer via OAuth 2.0 enables centralized user management. This guide covers the complete setup and all common failure points.

## Prerequisites

- Authentik running and accessible
- Portainer CE or BE installed
- Domain names for both services

## Step 1: Create an OAuth Application in Authentik

1. Log in to the Authentik Admin UI at `https://authentik.yourdomain.com/if/admin/`
2. Go to **Applications** → **Providers** → **Create**
3. Select **OAuth2/OpenID Provider**
4. Configure:
   - **Name**: Portainer
   - **Authorization flow**: default-authorization-flow
   - **Client type**: Confidential
   - **Client ID**: Authentik generates this (copy it)
   - **Client Secret**: Generate (copy it)
   - **Redirect URIs**: `https://portainer.yourdomain.com` (exact, no trailing slash)
   - **Scopes**: `openid email profile`

5. Create the Application:
   - **Name**: Portainer
   - **Provider**: Select the provider just created
   - **Launch URL**: `https://portainer.yourdomain.com`

## Step 2: Get Authentik OAuth Endpoints

```bash
# Authentik exposes OpenID Connect discovery endpoint
curl https://authentik.yourdomain.com/application/o/portainer/.well-known/openid-configuration | jq '{
  authorization_endpoint,
  token_endpoint,
  userinfo_endpoint
}'

# Typical values:
# authorization_endpoint: https://authentik.yourdomain.com/application/o/authorize/
# token_endpoint: https://authentik.yourdomain.com/application/o/token/
# userinfo_endpoint: https://authentik.yourdomain.com/application/o/userinfo/
```

## Step 3: Configure OAuth in Portainer

In Portainer BE (Business Edition):
1. Go to **Settings** → **Authentication**
2. Select **OAuth** as authentication method
3. Fill in:
   - **Client ID**: from Authentik
   - **Client secret**: from Authentik
   - **Authorization URL**: `https://authentik.yourdomain.com/application/o/authorize/`
   - **Access token URL**: `https://authentik.yourdomain.com/application/o/token/`
   - **Resource URL**: `https://authentik.yourdomain.com/application/o/userinfo/`
   - **Redirect URL**: `https://portainer.yourdomain.com`
   - **Logout URL**: `https://authentik.yourdomain.com/application/o/portainer/end-session/`
   - **Scopes**: `openid email profile`
   - **User identifier**: `email` or `preferred_username`
4. Click **Save settings**

## Step 4: Fix "Redirect URI Mismatch" Error

The most common OAuth error:

```
Error: redirect_uri_mismatch
The redirect URI in the request did not match the authorized redirect URIs.
```

Fix in Authentik:
1. Edit your Portainer OAuth provider
2. Ensure the redirect URI **exactly** matches what Portainer sends
3. Check for trailing slashes, http vs https, and www vs non-www

```bash
# Check what redirect URI Portainer is actually sending
# Enable debug logging and attempt login
docker logs portainer 2>&1 | grep -i "redirect\|oauth\|callback" | tail -10

# Common mismatch causes:
# Portainer sends: http://portainer.yourdomain.com (HTTP)
# Authentik expects: https://portainer.yourdomain.com (HTTPS)
# Fix: Ensure X-Forwarded-Proto header is set correctly in reverse proxy
```

## Step 5: Fix HTTPS Behind Reverse Proxy

Portainer must know it's behind HTTPS to generate the correct redirect URI:

```nginx
# Add to Nginx proxy config
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header Host $host;
```

Without this, Portainer generates `http://` redirect URLs even when accessed via HTTPS.

## Step 6: Fix Token Validation Errors

```bash
# Check Portainer logs for token errors
docker logs portainer 2>&1 | grep -i "token\|jwt\|invalid\|oauth" | tail -20

# Common issue: clock skew between Portainer and Authentik
# JWT tokens have 5-minute tolerance — if clocks differ more, tokens are rejected

# Sync time on Portainer host
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd
timedatectl status | grep NTP
```

## Step 7: Fix User Claim Mapping

Portainer needs specific claims from the userinfo endpoint:

```bash
# Test what Authentik returns for userinfo
# Get a test token from Authentik first, then:
curl -H "Authorization: Bearer YOUR_TEST_TOKEN" \
  https://authentik.yourdomain.com/application/o/userinfo/ | jq .

# Expected claims for Portainer:
# {
#   "sub": "user-uuid",
#   "email": "user@example.com",
#   "preferred_username": "username",
#   "name": "Full Name"
# }
```

If the `email` claim is missing, enable it in Authentik:
1. OAuth Provider → edit → Scopes → ensure `email` is included
2. Or add a custom property mapping

## Step 8: Configure Team Mapping

Map Authentik groups to Portainer teams (BE feature):

```bash
# In Portainer Settings → Authentication → OAuth
# Enable "Automatic Team Membership"
# Configure group claim name: "groups" (or "ak_groups" in Authentik)

# In Authentik, ensure groups are included in the token:
# Provider → edit → Property Mappings → add "authentik default OAuth Mapping: Groups"
```

## Step 9: Test OAuth Flow Manually

```bash
# Step 1: Get authorization code
# Open this URL in browser:
echo "https://authentik.yourdomain.com/application/o/authorize/?client_id=YOUR_CLIENT_ID&redirect_uri=https://portainer.yourdomain.com&response_type=code&scope=openid%20email%20profile"

# Step 2: Exchange code for token
# After redirect, you'll get: https://portainer.yourdomain.com?code=AUTHORIZATION_CODE
CODE="the_code_from_redirect"

curl -X POST \
  https://authentik.yourdomain.com/application/o/token/ \
  -d "grant_type=authorization_code" \
  -d "code=$CODE" \
  -d "redirect_uri=https://portainer.yourdomain.com" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET"
```

## Step 10: Fix Authentik Application Slug Mismatch

Authentik uses application slugs in URLs. Ensure the slug matches what you're using:

```bash
# Check Authentik application slug
# In Authentik: Applications → Application → copy the Slug

# The OAuth endpoints use the slug:
# https://authentik.yourdomain.com/application/o/SLUG/

# If endpoints return 404, the slug is wrong
curl -I https://authentik.yourdomain.com/application/o/portainer/.well-known/openid-configuration
# Should return 200
```

## Conclusion

OAuth integration between Portainer and Authentik most commonly fails due to redirect URI mismatches and X-Forwarded-Proto header issues. Always test the OAuth flow manually using `curl` to isolate exactly which step is failing. Ensure clocks are synchronized on both servers, all URIs use consistent HTTP/HTTPS schemes, and the userinfo endpoint returns the `email` or `preferred_username` claim that Portainer uses as the user identifier.
