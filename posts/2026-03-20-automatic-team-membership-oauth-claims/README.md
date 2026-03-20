# How to Configure Automatic Team Membership via OAuth Claims in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Teams, Claims, Business Edition

Description: Configure Portainer to automatically assign users to teams based on OAuth claims (groups) returned by your identity provider.

---

Portainer Business Edition can automatically assign users to teams based on group claims returned by your OAuth provider. This eliminates manual team management when user group memberships are already managed in your IdP.

## How Claim-Based Team Assignment Works

When a user logs in via OAuth:
1. Portainer calls the userinfo endpoint
2. The IdP returns user claims including group memberships
3. Portainer matches group names to existing team names
4. The user is added to matching teams automatically

## Configure Your IdP to Return Groups

### Azure AD / Entra ID

In Azure portal, configure the app to include group claims:

1. Go to **App registrations > [Your App] > Token configuration**
2. Click **Add groups claim**
3. Select **Security groups**
4. Choose to include group **Display names** (not GUIDs)

### Keycloak

Add a Group Membership mapper to your client (as covered in the Keycloak setup guide) with:
- **Token Claim Name**: `groups`
- **Full group path**: Disabled (return just group names, not `/path/to/group`)

### Authentik

In the OAuth provider's **Advanced Protocol Settings**, enable the `groups` scope.

## Configure Team Auto-Sync in Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Enable OAuth with group-based team assignment
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
      "TeamMembershipClaim": "groups"
    }
  }' \
  --insecure
```

## Create Portainer Teams Matching IdP Groups

Teams must exist in Portainer before the assignment can happen:

```bash
# Create teams that match your IdP group names EXACTLY
# (case-sensitive match)
for team in "devops" "developers" "platform-engineering" "qa-team"; do
  curl -s -X POST \
    https://localhost:9443/api/teams \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"Name\": \"$team\"}" \
    --insecure | python3 -c "
import sys, json
r = json.load(sys.stdin)
print(f'Created team: {r.get(\"Name\", \"?\")} (ID: {r.get(\"Id\", \"?\")})')
"
done
```

## Verify Claim-Based Assignment

After a user logs in, verify their team assignment:

```bash
# List users and their team memberships
curl -s https://localhost:9443/api/users \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -c "
import sys, json
users = json.load(sys.stdin)
for u in users:
    print(f'User: {u[\"Username\"]:<30} Teams: {u.get(\"TeamIDs\", [])}')
"
```

## Debugging Team Assignment

If users aren't being assigned to teams:

```bash
# Check Portainer debug logs after a login attempt
docker logs portainer 2>&1 | grep -i "team\|group\|claim\|oauth" | tail -20

# Verify the groups claim is actually returned by your IdP
# Get an access token and call userinfo manually
curl -H "Authorization: Bearer <access-token>" \
  https://idp.example.com/oauth/userinfo | python3 -m json.tool | grep groups
```

---

*Manage team-based access and monitor container health with [OneUptime](https://oneuptime.com).*
