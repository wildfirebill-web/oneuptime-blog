# How to Configure SSO (Single Sign-On) in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, SSO, Single Sign-On, OAuth, Authentication, Enterprise

Description: Enable SSO in Portainer so users are automatically logged in without the login screen when already authenticated with your identity provider.

## Introduction

SSO (Single Sign-On) in Portainer means users who are already authenticated with their identity provider (via Azure AD, Okta, Google, etc.) are automatically logged into Portainer without seeing a login screen. The SSO feature in Portainer works by immediately initiating the OAuth flow when users visit the Portainer URL, skipping the login page entirely.

## Prerequisites

- OAuth authentication configured in Portainer
- Identity provider already set up
- Users already authenticated with the IdP in their browser session

## Understanding SSO vs. OAuth in Portainer

| Mode | Behavior |
|------|---------|
| OAuth only | User sees Portainer login page with an "OAuth Login" button |
| SSO enabled | User is automatically redirected to IdP (no login page shown) |

SSO is an enhancement on top of OAuth — it's only relevant once OAuth is configured.

## Enabling SSO via UI

1. Go to Settings → Authentication → OAuth
2. Configure your OAuth provider
3. Find the **SSO** toggle
4. Enable it
5. Save settings

Once enabled, visiting `https://portainer.example.com/` will immediately redirect to your IdP if the user isn't logged in.

## Enabling SSO via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Enable SSO in OAuth settings
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "AuthenticationMethod": 3,
    "oauthsettings": {
      "ClientID": "your-client-id",
      "ClientSecret": "your-client-secret",
      "AuthorizationURI": "https://your-idp.com/oauth/authorize",
      "AccessTokenURI": "https://your-idp.com/oauth/token",
      "ResourceURI": "https://your-idp.com/oauth/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "email",
      "Scopes": "openid profile email",
      "OAuthAutoCreateUsers": true,
      "SSO": true,
      "HideInternalAuth": true
    }
  }'
```

## How SSO Works Step by Step

```
1. User visits https://portainer.example.com/
2. Portainer detects no active session
3. (SSO enabled) Portainer immediately redirects to:
   https://idp.example.com/oauth/authorize?client_id=...&redirect_uri=...
4. IdP checks if user has active session
   a. If yes: IdP issues tokens and redirects back to Portainer
   b. If no: IdP shows login page, user authenticates, then redirects back
5. Portainer validates tokens and creates user session
6. User sees Portainer dashboard
```

## Hiding the Internal Login (HideInternalAuth)

When SSO is enabled, you can hide the internal username/password form:

```json
"HideInternalAuth": true
```

With this enabled, the login page only shows the OAuth/SSO button. To access internal admin account:

1. Append `?skipSSO=true` to the Portainer URL: `https://portainer.example.com/?skipSSO=true`
2. This shows the internal login form even with SSO enabled

**Important**: Always keep at least one internal admin account with a known password for emergency access in case the IdP is unavailable.

## Logout URI Configuration

Configure where users are redirected after logging out of Portainer:

```json
"LogoutURI": "https://idp.example.com/oauth/logout?redirect_uri=https://portainer.example.com/"
```

This ensures the IdP session is also terminated when users log out of Portainer.

## SSO with Azure AD

Azure AD supports silent authentication if the user has an active SSO session. When a user has already logged into Office 365, visiting Portainer with SSO enabled:

1. Portainer redirects to Azure AD
2. Azure AD detects existing session (no re-authentication needed)
3. Azure AD redirects back to Portainer with tokens
4. User is logged in automatically

## Conclusion

SSO in Portainer creates a seamless experience for users who already have an active session with the corporate IdP — they simply visit the Portainer URL and are logged in immediately. Combined with `HideInternalAuth`, the Portainer login page is invisible to regular users. Always maintain an emergency internal admin account accessible via `?skipSSO=true` for operational continuity when the IdP is unavailable.
