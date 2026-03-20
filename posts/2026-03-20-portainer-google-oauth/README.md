# How to Configure Google OAuth with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Google, OAuth, SSO, Authentication, Google Workspace

Description: Set up Google OAuth 2.0 authentication in Portainer to allow users to sign in with their Google or Google Workspace accounts.

## Introduction

Google OAuth allows Portainer users to authenticate using their Google or Google Workspace (G Suite) accounts. This is ideal for organizations using Google Workspace as their identity provider or for teams who want to use their Google accounts for Portainer access.

## Prerequisites

- Google Cloud project (free to create)
- Portainer running on HTTPS
- A registered redirect URI in Google Cloud Console

## Step 1: Create Google OAuth Credentials

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Select or create a project
3. Navigate to **APIs & Services** → **Credentials**
4. Click **Create Credentials** → **OAuth client ID**
5. Select **Web application** as the application type

Fill in:
```
Name:             Portainer
Authorized redirect URIs:
  https://portainer.example.com/
```

6. Click **Create**
7. Note the **Client ID** and **Client Secret**

## Step 2: Configure the OAuth Consent Screen

1. Go to **APIs & Services** → **OAuth consent screen**
2. For internal use, select **Internal** (limits to your Google Workspace org)
3. Fill in application name: "Portainer"
4. Add required scopes: `email`, `profile`, `openid`
5. Save

## Step 3: Configure Portainer for Google OAuth

Google's OIDC endpoints:

```
Authorization URL: https://accounts.google.com/o/oauth2/v2/auth
Access Token URL:  https://oauth2.googleapis.com/token
Resource URL:      https://openidconnect.googleapis.com/v1/userinfo
```

In Settings → Authentication → OAuth → Google:

```
Client ID:         your-client-id.apps.googleusercontent.com
Client Secret:     your-client-secret
Authorization URL: https://accounts.google.com/o/oauth2/v2/auth
Access Token URL:  https://oauth2.googleapis.com/token
Resource URL:      https://openidconnect.googleapis.com/v1/userinfo
Redirect URL:      https://portainer.example.com/
User Identifier:   email
Scopes:            openid email profile
```

## Step 4: Configure via API

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d '{
    "AuthenticationMethod": 3,
    "oauthsettings": {
      "ClientID": "123456789-abcdefg.apps.googleusercontent.com",
      "ClientSecret": "GOCSPX-your_secret",
      "AuthorizationURI": "https://accounts.google.com/o/oauth2/v2/auth",
      "AccessTokenURI": "https://oauth2.googleapis.com/token",
      "ResourceURI": "https://openidconnect.googleapis.com/v1/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "email",
      "Scopes": "openid email profile",
      "OAuthAutoCreateUsers": true,
      "SSO": true
    }
  }'
```

## Restricting Access to Specific Domains (Google Workspace)

Add the `hd` parameter to the authorization URL to restrict login to your organization's domain:

```
Authorization URL: https://accounts.google.com/o/oauth2/v2/auth?hd=yourcompany.com
```

This ensures only users with `@yourcompany.com` Google accounts can log in.

## Restricting Access to Specific Users

In Google Cloud Console:
1. Set the consent screen to **Internal** — only your Workspace users can log in
2. If you need to allow specific external emails, add them as test users during development

## Verifying the Configuration

```bash
# Manually test the OAuth flow
# Step 1: Generate the authorization URL
CLIENT_ID="your-client-id.apps.googleusercontent.com"
REDIRECT_URI="https://portainer.example.com/"
STATE=$(openssl rand -hex 16)

echo "Visit this URL to test:"
echo "https://accounts.google.com/o/oauth2/v2/auth?client_id=${CLIENT_ID}&redirect_uri=${REDIRECT_URI}&response_type=code&scope=openid+email+profile&state=${STATE}"
```

## Troubleshooting

**"redirect_uri_mismatch"**: The redirect URI in Portainer doesn't match what's registered in Google Cloud Console. Check for trailing slashes and exact URL match.

**"access_denied"**: The user declined consent or the consent screen isn't configured correctly.

**"invalid_client"**: Wrong Client ID or Secret. Verify them in Google Cloud Console.

## Conclusion

Google OAuth provides a simple SSO experience for teams already using Google accounts. The setup requires only a few minutes in Google Cloud Console and Portainer's settings. For Google Workspace organizations, the `hd` parameter domain restriction ensures only corporate accounts can access Portainer — a simple but effective access control measure.
