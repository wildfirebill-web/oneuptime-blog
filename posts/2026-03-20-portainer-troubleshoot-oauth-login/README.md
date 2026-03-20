# How to Troubleshoot OAuth Login Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Troubleshooting, SSO, Authentication, Debugging

Description: Diagnose and fix common OAuth authentication failures in Portainer including redirect URI mismatches, token errors, and configuration issues.

## Introduction

OAuth authentication issues in Portainer range from simple configuration mismatches to complex token parsing problems. This guide provides a systematic approach to diagnosing OAuth failures, with specific error messages and their fixes.

## Common OAuth Error Messages

### "redirect_uri_mismatch"

The redirect URI in Portainer doesn't match what's registered with the IdP.

```bash
# Check current Portainer redirect URI setting
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/settings \
  | python3 -c "import sys,json; s=json.load(sys.stdin); o=s.get('oauthsettings',{}); print('Redirect URI:', o.get('RedirectURI',''))"

# Fix: Ensure exact match including trailing slash
# Portainer: https://portainer.example.com/
# IdP:       https://portainer.example.com/ (must be identical)
```

### "invalid_client"

Client ID or secret is wrong.

```bash
# Verify client credentials with IdP directly
curl -X POST \
  "https://your-idp.com/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=your-client-id&client_secret=your-client-secret&grant_type=client_credentials"

# If this fails, your client ID or secret is wrong
```

### "Origin is not trusted" (after OAuth redirect)

Portainer CSRF protection rejects the OAuth callback.

```bash
# Fix: Add trusted origins
docker exec portainer grep -i "trusted" /proc/1/cmdline || \
docker inspect portainer | python3 -c "import sys,json; c=json.load(sys.stdin); print(c[0]['Config']['Cmd'])"

# Add --trusted-origins flag if missing
# This must include the HTTPS URL users access Portainer through
```

### "Token parsing failed" or blank page after redirect

The ID token or userinfo response doesn't contain the expected user identifier claim.

```bash
# Debug: Check what claims are in the token
# Method 1: Check Portainer logs
docker logs portainer 2>&1 | grep -i "oauth\|token\|claim" | tail -20

# Method 2: Manually call the userinfo endpoint
OAUTH_TOKEN="access-token-from-idp"
curl -H "Authorization: Bearer $OAUTH_TOKEN" \
  "https://your-idp.com/oauth/userinfo" | python3 -m json.tool

# Verify the configured "User Identifier" claim exists in the response
# Common values: sub, email, preferred_username, login
```

## Diagnostic Checklist

```bash
# 1. Check OAuth settings are saved
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/settings \
  | python3 -c "
import sys, json
s = json.load(sys.stdin)
o = s.get('oauthsettings', {})
print('Auth Method:', s.get('AuthenticationMethod'))  # Should be 3
print('Client ID:', o.get('ClientID', 'NOT SET'))
print('Auth URI:', o.get('AuthorizationURI', 'NOT SET'))
print('Token URI:', o.get('AccessTokenURI', 'NOT SET'))
print('User Info URI:', o.get('ResourceURI', 'NOT SET'))
print('Redirect URI:', o.get('RedirectURI', 'NOT SET'))
print('User Identifier:', o.get('UserIdentifier', 'NOT SET'))
print('Scopes:', o.get('Scopes', 'NOT SET'))
"
```

## Network-Level Debugging

```bash
# Check if Portainer can reach the IdP endpoints
docker exec portainer wget -q --spider \
  "https://your-idp.com/.well-known/openid-configuration" 2>&1

# Or use curl if wget not available
docker run --rm --network container:portainer \
  alpine/curl -sv \
  "https://your-idp.com/.well-known/openid-configuration" 2>&1 | head -30

# Check DNS resolution for IdP
docker exec portainer nslookup your-idp.com
```

## Browser-Level Debugging

When OAuth fails silently, use browser developer tools:

1. Open DevTools → Network tab
2. Click the OAuth login button
3. Watch the redirects
4. Look for the request to your IdP and the redirect back to Portainer
5. Check the callback URL parameters for error codes

Common callback parameters on error:
```
error=redirect_uri_mismatch
error_description=...
```

## Portainer Log Analysis

```bash
# Enable verbose logging to see OAuth flow
docker logs portainer -f 2>&1 | grep -i "oauth\|auth\|token\|error"

# Save recent logs for analysis
docker logs portainer --since 10m 2>&1 > portainer-auth-debug.log
grep -i "oauth\|401\|403\|error" portainer-auth-debug.log
```

## Testing OAuth Manually

Test the complete OAuth flow without Portainer:

```bash
# Step 1: Get authorization URL
CLIENT_ID="your-client-id"
REDIRECT_URI="https://portainer.example.com/"
SCOPE="openid profile email"
STATE=$(openssl rand -hex 16)

AUTH_URL="https://your-idp.com/oauth/authorize?client_id=${CLIENT_ID}&redirect_uri=${REDIRECT_URI}&response_type=code&scope=${SCOPE}&state=${STATE}"
echo "Test URL: $AUTH_URL"
# Visit this URL in a browser and note the code parameter in the callback URL

# Step 2: Exchange code for token
CODE="code-from-callback"
curl -X POST "https://your-idp.com/oauth/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&client_id=${CLIENT_ID}&client_secret=YOUR_SECRET&redirect_uri=${REDIRECT_URI}&code=${CODE}"

# Step 3: Get user info
ACCESS_TOKEN="access-token-from-step2"
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://your-idp.com/oauth/userinfo"
```

## Conclusion

OAuth troubleshooting requires checking both the Portainer side (configuration, logs, network access) and the IdP side (client registration, redirect URIs, token claims). The most common issue is redirect URI mismatch — fix it by ensuring character-for-character identity between Portainer's setting and the IdP's registered URI. For harder issues, manual OAuth flow testing isolates whether the problem is in the IdP configuration or Portainer's token handling.
