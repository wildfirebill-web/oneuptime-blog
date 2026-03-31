# How to Configure Automatic Team Membership via OAuth Claims in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OAuth, Team, Claims, RBAC, Business Edition, Automation

Description: Use OAuth token claims to automatically assign users to Portainer teams based on their identity provider group membership.

## Introduction

Portainer Business Edition can read group membership claims from OAuth tokens and automatically assign users to matching Portainer teams. This eliminates manual team management - changes in your IdP groups are reflected in Portainer at the user's next login.

## Prerequisites

- Portainer Business Edition
- OAuth authentication configured
- IdP configured to include group claims in tokens
- Portainer teams created with names matching IdP group names

## Step 1: Configure Your IdP to Include Group Claims

### Azure AD (Entra ID)

1. In the Azure App Registration → **Token configuration**
2. Click **Add groups claim**
3. Select **Security groups** (and/or **Directory roles**)
4. Under **ID token**: enable group claims
5. Optionally, under **Advanced settings**, set **Customize token properties by type** and set the claim to emit group display names instead of IDs

```json
// Example ID token payload with groups claim
{
  "sub": "...",
  "email": "alice@corp.com",
  "groups": [
    "portainer-devops",
    "all-employees"
  ]
}
```

### Keycloak

Add a Group Membership mapper to the client scope:

```text
Mapper Type:       Group Membership
Token Claim Name:  groups
Full group path:   Off
Add to userinfo:   On
```

### Authelia

Enable groups scope:
```yaml
clients:
  - id: portainer
    scopes:
      - openid
      - profile
      - email
      - groups  # Include this
```

### Authentik

Create a scope mapping that includes groups:

1. **Customisation** → **Property Mappings**
2. Create a new **Scope Mapping**:

```python
# Name: Portainer Groups

# Scope name: groups
return {
    "groups": [group.name for group in request.user.ak_groups.all()]
}
```

## Step 2: Verify Claims Are Present

Test your OAuth flow and inspect the token:

```bash
# Decode the JWT token (base64 decode the middle part)
JWT_TOKEN="eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIxMjM0In0.signature"
echo $JWT_TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

Verify the `groups` claim is present and contains the expected group names.

## Step 3: Configure Claim Name in Portainer

In Settings → Authentication → OAuth (Portainer BE):

```text
Team membership claim:   groups
```

This tells Portainer which claim in the token contains group names.

## Step 4: Create Teams Matching Group Names

Create Portainer teams with names that exactly match the group names in the OAuth claim:

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create teams matching OAuth group names
GROUPS=("portainer-devops" "portainer-qa" "portainer-readonly" "portainer-admins")

for group in "${GROUPS[@]}"; do
  curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    https://portainer.example.com/api/teams \
    -d "{\"name\": \"${group}\"}"
  echo "Created team: $group"
done
```

## Step 5: Assign Environment Access to Teams

After teams are created, grant them environment access:

```bash
# Get team IDs
TEAMS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/teams \
  | python3 -c "import sys,json; [print(f'{t[\"Id\"]}:{t[\"Name\"]}') for t in json.load(sys.stdin)]")

echo "$TEAMS"

# Assign team 1 (devops) to environment 1 with Standard User role
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/teamaccesspolicies \
  -d '{"1": {"RoleId": 2}}'
```

## Verifying Team Synchronization

1. Log in with an OAuth user who belongs to a group
2. Log out and log back in
3. Check the user's team memberships via API:

```bash
# Get user ID for the OAuth user
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/users \
  | python3 -c "import sys,json; [print(f'ID={u[\"Id\"]} User={u[\"Username\"]}') for u in json.load(sys.stdin)]"

# Check team memberships for user ID 3
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/users/3/memberships \
  | python3 -m json.tool
```

## Conclusion

OAuth claims-based team membership in Portainer Business Edition creates a fully automated user lifecycle - users' Portainer team memberships mirror their IdP group memberships and update on every login. The critical requirements are: IdP configured to include group names (not IDs) in tokens, the correct claim name configured in Portainer, and Portainer teams created with matching names.
