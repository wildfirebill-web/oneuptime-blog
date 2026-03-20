# How to Configure Google OAuth with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Google OAuth, SSO, Authentication, Business Edition

Description: Configure Google OAuth to allow users to sign into Portainer with their Google Workspace or personal Google accounts.

---

Google OAuth integration lets users log in to Portainer using their Google accounts. For organizations using Google Workspace, this enables single sign-on without managing separate credentials.

## Step 1: Create a Google OAuth Application

1. Go to the [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project or select an existing one
3. Navigate to **APIs & Services > OAuth consent screen**
4. Choose **Internal** (Google Workspace only) or **External** (any Google account)
5. Fill in the app name, support email, and authorized domains
6. Navigate to **APIs & Services > Credentials**
7. Click **Create Credentials > OAuth client ID**
8. Select **Web application**
9. Add the authorized redirect URI: `https://portainer.example.com/`
10. Click **Create** and copy the **Client ID** and **Client Secret**

## Step 2: Configure Portainer with Google OAuth

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 3,
    "OAuthSettings": {
      "ClientID": "123456789.apps.googleusercontent.com",
      "ClientSecret": "GOCSPX-your-client-secret",
      "AuthorizationURI": "https://accounts.google.com/o/oauth2/v2/auth",
      "AccessTokenURI": "https://oauth2.googleapis.com/token",
      "ResourceURI": "https://www.googleapis.com/oauth2/v3/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "email",
      "Scopes": "openid email profile",
      "OAuthAutoCreateUsers": true
    }
  }' \
  --insecure
```

## Google OAuth Endpoint Reference

| Field | Value |
|-------|-------|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Access Token URL | `https://oauth2.googleapis.com/token` |
| Resource URL | `https://www.googleapis.com/oauth2/v3/userinfo` |
| User Identifier | `email` |
| Scopes | `openid email profile` |

## Restrict to Google Workspace Domain

To allow only users from your company's Google Workspace domain, add the `hd` (hosted domain) parameter. This is done via the authorization URL in some configurations, or by checking the `hd` claim in the returned token:

```bash
# Add domain restriction to scopes or use Portainer team mapping
# The Google userinfo response includes "hd" field for Workspace accounts
# Portainer can use "email" as the identifier, then you manually verify domain

# To enforce domain restriction, only allow auto-create for your domain:
# In Portainer, after setup, disable auto-create and manually approve users
# or use your IdP's built-in access controls
```

## Configure via Portainer UI

1. Navigate to **Settings > Authentication**
2. Select **OAuth > Google**
3. Portainer may have pre-filled Google-specific endpoints
4. Enter your **Client ID** and **Client Secret**
5. Set **Redirect URL** to your Portainer URL with trailing slash
6. Click **Save settings**

## Test the Google OAuth Login

1. Open Portainer in a private/incognito browser tab
2. Click **Login with OAuth** (or the Google button)
3. Choose your Google account
4. Grant the requested permissions
5. You should be redirected back to Portainer

---

*Manage container uptime with [OneUptime](https://oneuptime.com) alongside your Google-authenticated Portainer.*
