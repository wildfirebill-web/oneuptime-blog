# How to Set Up Auto-Admin Assignment for OAuth Groups in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Admin, RBAC, Business Edition

Description: Configure Portainer to automatically grant administrator privileges to users who belong to a specific OAuth group or claim.

---

Instead of manually promoting OAuth users to administrators, Portainer BE can automatically grant admin role to users whose IdP groups match a configured admin group name.

## How Admin Auto-Assignment Works

When a user logs in via OAuth:
1. Portainer checks the `groups` claim in the userinfo response
2. If the user's groups include the configured admin group, they are granted the Administrator role
3. If the user is removed from the admin group in the IdP, they lose admin access on next login

## Configure Admin Group in Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure OAuth with admin group assignment
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 3,
    "OAuthSettings": {
      "ClientID": "portainer",
      "ClientSecret": "client-secret",
      "AuthorizationURI": "https://idp.example.com/oauth/authorize",
      "AccessTokenURI": "https://idp.example.com/oauth/token",
      "ResourceURI": "https://idp.example.com/oauth/userinfo",
      "RedirectURI": "https://portainer.example.com/",
      "UserIdentifier": "email",
      "Scopes": "openid profile email groups",
      "OAuthAutoCreateUsers": true,
      "OAuthTeamMemberships": true,
      "TeamMembershipClaim": "groups",
      "AdminGroupName": "portainer-admins"
    }
  }' \
  --insecure
```

## Set Up the Admin Group in Your IdP

### Azure AD

```powershell
# Create a security group for Portainer admins
New-AzureADGroup -DisplayName "portainer-admins" -MailEnabled $false -SecurityEnabled $true -MailNickName "portainer-admins"

# Add admin users
Add-AzureADGroupMember -ObjectId "<group-id>" -RefObjectId "<user-id>"
```

### Keycloak

1. Navigate to **Groups** in your realm
2. Create a group named `portainer-admins`
3. Add users who should have admin access
4. Ensure the group name claim mapper returns this group name

### Testing the Admin Group Claim

```bash
# After getting an OAuth token, check the groups returned
curl -H "Authorization: Bearer <access-token>" \
  https://idp.example.com/userinfo | python3 -m json.tool

# Look for output like:
# {
#   "email": "admin@example.com",
#   "groups": ["portainer-admins", "developers"],
#   ...
# }
```

## Verify Admin Assignment

```bash
# After an admin group member logs in, check their role
curl -s https://localhost:9443/api/users \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -c "
import sys, json
users = json.load(sys.stdin)
for u in users:
    role = 'Admin' if u.get('Role') == 1 else 'User'
    print(f'{u[\"Username\"]:<40} Role: {role}')
"
```

## Security Considerations

- The `AdminGroupName` setting is case-sensitive — ensure exact match with the IdP group name
- Use a dedicated IdP group for Portainer admins, separate from general IT admin groups
- Regularly audit IdP group membership for the admin group
- Keep at least one local admin account active as a fallback

---

*Monitor admin actions and container events with [OneUptime](https://oneuptime.com) observability.*
