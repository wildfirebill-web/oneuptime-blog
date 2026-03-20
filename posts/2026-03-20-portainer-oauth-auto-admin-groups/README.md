# How to Set Up Auto-Admin Assignment for OAuth Groups in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Admin, Groups, Automation, RBAC

Description: Automatically assign administrator privileges to Portainer users who belong to specific OAuth identity provider groups.

## Introduction

Manually assigning admin roles to users after OAuth login creates management overhead. Portainer Business Edition supports automatic admin role assignment based on group membership in your OAuth identity provider. Members of a designated "admin group" receive Portainer administrator privileges automatically on login.

## Prerequisites

- Portainer Business Edition
- OAuth authentication configured
- IdP configured to include group claims in tokens
- Portainer team with the admin group name created

## How Auto-Admin Assignment Works

1. User logs in via OAuth
2. Portainer reads the `groups` claim (or configured claim name)
3. If any group matches the configured "admin group", user gets admin role
4. If no admin group matches, user gets standard user role
5. Role is re-evaluated on each login

## Step 1: Create Admin Group in IdP

### Azure AD

```powershell
# Create a security group for Portainer admins

New-AzureADGroup -DisplayName "portainer-admins" `
  -MailEnabled $false `
  -SecurityEnabled $true `
  -MailNickName "portainer-admins"

# Add admin users to the group
Add-AzureADGroupMember -ObjectId (Get-AzureADGroup -SearchString "portainer-admins").ObjectId `
  -RefObjectId (Get-AzureADUser -UserPrincipalName "alice@corp.com").ObjectId
```

### Keycloak

```bash
KEYCLOAK_URL="https://keycloak.example.com"
REALM="myrealm"

# Create the admin group
curl -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "${KEYCLOAK_URL}/admin/realms/${REALM}/groups" \
  -d '{"name": "portainer-admins"}'
```

### Authentik

Create a group named `portainer-admins` in Authentik's admin UI under **Directory** → **Groups**.

## Step 2: Ensure Admin Group is in Token Claims

Verify the admin group name appears in the OAuth token's groups claim:

```bash
# Decode a test token to verify group names
# (Get a token by logging in and capturing it, or use IdP testing tools)
echo "eyJ..." | cut -d'.' -f2 | base64 -d | python3 -m json.tool | grep -A10 '"groups"'

# Expected output:
# "groups": ["portainer-admins", "all-employees", "it-department"]
```

## Step 3: Configure Auto-Admin in Portainer

### Via Web UI (Portainer BE)

1. Go to Settings → Authentication → OAuth
2. Under **Team settings**, find **Admin team/group claim**
3. Enter the group name: `portainer-admins`
4. Save settings

### Via API

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
      "ClientID": "your-client-id",
      "ClientSecret": "your-client-secret",
      "AuthorizationURI": "https://idp.example.com/oauth/authorize",
      "AccessTokenURI": "https://idp.example.com/oauth/token",
      "ResourceURI": "https://idp.example.com/oauth/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "email",
      "Scopes": "openid profile email groups",
      "OAuthAutoCreateUsers": true,
      "SSO": true,
      "TeamMemberships": {
        "OAuthClaimName": "groups",
        "OAuthClaimMatchers": [
          {
            "ClaimValRegex": "portainer-admins",
            "RoleId": 1,
            "TeamId": 0
          }
        ]
      }
    }
  }'
```

## Step 4: Test Admin Assignment

1. Log in with an account that's a member of `portainer-admins`
2. After login, check if you see the admin menu items (Settings, Users, Teams, etc.)
3. Verify via API:

```bash
# The admin user's token should allow admin API calls
ALICE_TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"alicepassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Try an admin-only endpoint
curl -s -H "Authorization: Bearer $ALICE_TOKEN" \
  https://portainer.example.com/api/users \
  | python3 -m json.tool
# Should return list of users (admin only endpoint)
```

## Security Considerations

- Keep the admin group small and well-governed
- Review admin group membership regularly in your IdP
- Remove admin group membership (not just Portainer access) for users who leave admin roles
- Consider naming the admin group something specific to Portainer (e.g., `portainer-admins`) rather than using a generic IT-admins group that includes more people than need Portainer admin access

## Conclusion

Auto-admin assignment based on OAuth groups centralizes role management in your identity provider. Granting or revoking Portainer admin privileges is now a matter of adding or removing users from the admin group in your IdP - no Portainer UI interaction required. This is particularly valuable in organizations where the IdP team manages group membership independently from the infrastructure team managing Portainer.
