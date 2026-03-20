# How to Hide Internal Authentication When Using OAuth in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, SSO, Security, Authentication

Description: Configure Portainer to hide the internal username/password login form and show only the OAuth SSO button for a cleaner login experience.

---

Once OAuth SSO is configured and working reliably, you may want to hide the internal Portainer username/password form to enforce SSO-only authentication and provide a simpler login experience.

## Prerequisites

- OAuth authentication configured and verified working
- At least one local admin account as an emergency fallback
- Portainer Business Edition

## Enable Hide Internal Authentication

### Via the UI

1. Navigate to **Settings > Authentication**
2. Enable **OAuth** authentication
3. Find the **Hide internal authentication prompt** toggle
4. Enable it and click **Save**

### Via the API

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Enable OAuth and hide the internal login form
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 3,
    "OAuthSettings": {
      "ClientID": "your-client-id",
      "ClientSecret": "your-client-secret",
      "AuthorizationURI": "https://idp.example.com/oauth/authorize",
      "AccessTokenURI": "https://idp.example.com/oauth/token",
      "ResourceURI": "https://idp.example.com/oauth/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "email",
      "Scopes": "openid profile email",
      "OAuthAutoCreateUsers": true,
      "HideInternalAuth": true
    }
  }' \
  --insecure
```

## Emergency Access When OAuth is Hidden

When `HideInternalAuth` is true, the internal login form is hidden but can be accessed via a direct URL:

```
https://portainer.example.com/#!/auth
```

Some versions use a query parameter to bypass the SSO redirect:

```
https://portainer.example.com/?auth=local
```

## Keep a Local Admin Account Active

Before hiding internal authentication, ensure you have a working local admin account:

```bash
# Verify local admin account exists and can authenticate
curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"emergency-admin","password":"strongpassword123!"}' \
  --insecure | python3 -c "
import sys, json
r = json.load(sys.stdin)
if 'jwt' in r:
    print('Emergency admin login: SUCCESS')
else:
    print(f'Emergency admin login: FAILED - {r}')
"
```

## What Users See

After enabling `HideInternalAuth`:
- The login page shows only the **Login with OAuth** button
- No username/password fields are displayed
- Users are automatically redirected to the IdP if clicking the SSO button

## Re-Enable Internal Authentication

If you need to re-enable the internal login form:

```bash
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"OAuthSettings": {"HideInternalAuth": false}}' \
  --insecure
```

---

*Ensure your SSO-protected Portainer stays available with monitoring from [OneUptime](https://oneuptime.com).*
