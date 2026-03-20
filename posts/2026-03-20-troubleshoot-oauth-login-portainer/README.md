# How to Troubleshoot OAuth Login Issues in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Troubleshooting, Authentication, Debugging

Description: Diagnose and resolve common OAuth authentication failures in Portainer including redirect errors, token issues, and provider misconfigurations.

---

OAuth login failures in Portainer can have many causes — from misconfigured redirect URIs to expired client secrets. This guide provides systematic troubleshooting steps.

## Step 1: Enable Debug Logging

```bash
# Restart Portainer with debug logging
docker stop portainer && docker container rm portainer

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --log-level DEBUG

# Monitor OAuth-related log entries
docker logs -f portainer 2>&1 | grep -i "oauth\|token\|auth\|redirect"
```

## Common Error: redirect_uri_mismatch

This is the most frequent OAuth error:

```
Error: redirect_uri_mismatch
The redirect URI in the request did not match a registered redirect URI.
```

**Fix:**

```bash
# Check what URI Portainer is sending
docker logs portainer 2>&1 | grep -i "redirect"

# Verify the OAuthSettings.RedirectURI in Portainer
TOKEN=$(curl -s -X POST https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpass"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -s https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" --insecure | \
  python3 -c "import sys,json; s=json.load(sys.stdin); print(s.get('OAuthSettings',{}).get('RedirectURI','not set'))"

# Update to match exactly what's in your OAuth provider
curl -X PUT https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"OAuthSettings": {"RedirectURI": "https://portainer.example.com/"}}' \
  --insecure
```

## Common Error: invalid_client

```
Error: invalid_client
Client authentication failed.
```

**Fix:** Your client secret is wrong or expired:
```bash
# Re-enter the client secret (regenerate in your IdP if needed)
curl -X PUT https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"OAuthSettings": {"ClientSecret": "new-client-secret"}}' \
  --insecure
```

## Common Error: User Not Created After Login

User authenticates but Portainer shows "user not found":

```bash
# Check if AutoCreateUsers is enabled
curl -s https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" --insecure | \
  python3 -c "import sys,json; s=json.load(sys.stdin); print('AutoCreate:', s.get('OAuthSettings',{}).get('OAuthAutoCreateUsers', False))"

# Enable auto-create
curl -X PUT https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"OAuthSettings": {"OAuthAutoCreateUsers": true}}' \
  --insecure
```

## Common Error: Userinfo Endpoint Fails

```bash
# Test the userinfo endpoint manually with a token from your IdP
ACCESS_TOKEN="eyJ..."

curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://idp.example.com/oauth/userinfo

# Check Portainer debug logs for the exact URL it's calling
docker logs portainer 2>&1 | grep -i "resource\|userinfo"
```

## OAuth Troubleshooting Checklist

- [ ] Client ID matches exactly (no leading/trailing spaces)
- [ ] Client secret is current and valid
- [ ] Redirect URI matches exactly including trailing slash
- [ ] Authorization URL is correct for your IdP
- [ ] Access Token URL is correct
- [ ] Resource/Userinfo URL is accessible from Portainer container
- [ ] Scopes include `openid` if using OIDC
- [ ] UserIdentifier matches a claim actually returned by the IdP
- [ ] OAuth provider has granted admin consent if required (Azure AD)

---

*Detect OAuth-related outages early with [OneUptime](https://oneuptime.com) monitoring and alerting.*
