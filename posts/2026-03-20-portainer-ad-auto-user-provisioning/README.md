# How to Set Up Automatic User Provisioning with Active Directory in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Active Directory, User Provisioning, Automation, LDAP, Enterprise

Description: Configure Portainer to automatically create user accounts and team memberships for Active Directory users on their first login.

## Introduction

Manual user provisioning - creating Portainer accounts for each employee - doesn't scale. Portainer's automatic user provisioning creates accounts automatically when AD users log in for the first time, optionally placing them in teams based on their AD group membership. This guide covers the full automatic provisioning setup.

## How Automatic Provisioning Works

1. **First Login**: User authenticates with AD credentials
2. **Account Creation**: Portainer creates a local user account linked to their AD identity
3. **Team Assignment**: Portainer checks the user's AD groups and adds them to matching teams
4. **Access Granted**: User can access environments assigned to their team(s)
5. **Subsequent Logins**: Group membership is re-synchronized each time

## Step 1: Enable Auto-Create Users

In Settings → Authentication → LDAP:

1. Enable **Automatic user provisioning**
2. Optionally select a **Default team** for all new LDAP users

Or via API:

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
    "AuthenticationMethod": 2,
    "ldapsettings": {
      "Servers": [
        {
          "Host": "dc01.corp.example.com",
          "Port": 389,
          "ReaderDN": "portainer-svc@corp.example.com",
          "Password": "ServicePassword123"
        }
      ],
      "SearchSettings": [
        {
          "BaseDN": "DC=corp,DC=example,DC=com",
          "Username": "sAMAccountName",
          "Filter": "(&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
        }
      ],
      "GroupSearchSettings": [
        {
          "GroupBaseDN": "DC=corp,DC=example,DC=com",
          "GroupAttribute": "memberOf",
          "GroupFilter": "(objectClass=group)"
        }
      ],
      "AutoCreateUsers": true
    }
  }'
```

## Step 2: Set Up the Team Structure

Pre-create teams in Portainer that correspond to AD groups:

```bash
# Create teams that match AD group names

AD_GROUPS=("IT-Admins" "Developers" "QA-Engineers" "Operations" "Security-Team")

for group in "${AD_GROUPS[@]}"; do
  curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    https://portainer.example.com/api/teams \
    -d "{\"name\": \"${group}\"}"
done
```

## Step 3: Configure Environment Access for Teams

Assign each team appropriate access to environments:

```bash
# Get all teams
TEAMS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/teams)

# Get all environments
ENDPOINTS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints)

# Example: Give IT-Admins full access to all environments
# (Run this for each environment)
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/teamaccesspolicies \
  -d '{
    "1": {"RoleId": 2},
    "2": {"RoleId": 2},
    "3": {"RoleId": 1}
  }'
```

## Step 4: Test Automatic Provisioning

```bash
# Test login with an AD user account
curl -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"AlicePassword123"}' \
  | python3 -m json.tool

# If successful, check the created user
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/users \
  | python3 -c "import sys,json; [print(f'ID={u[\"Id\"]} User={u[\"Username\"]}') for u in json.load(sys.stdin)]"
```

## Understanding What Gets Created vs Managed

| Item | Created At | Updated At |
|------|-----------|-----------|
| User account | First login | Never (static) |
| Team membership | Each login | Each login |
| Environment access | Manually by admin | Manually by admin |
| User role (admin/user) | First login (default: user) | Manually or via AD admin group |

## Setting Admin Role via AD Group

To automatically give admin role to members of a specific AD group:

In LDAP settings, configure the admin attribute:
```text
Admin group: CN=Portainer-Admins,OU=Groups,DC=corp,DC=example,DC=com
```

Members of `Portainer-Admins` in AD will be provisioned as Portainer administrators.

## Deprovisioning: Handling AD User Removals

Portainer does not automatically delete users when removed from AD. To deprovision:

1. Delete or disable the user in AD (they can no longer log in)
2. Remove their Portainer account via UI or API to clean up

```bash
# Find and delete inactive users
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/users \
  | python3 -c "import sys,json; [print(f'{u[\"Id\"]}:{u[\"Username\"]}') for u in json.load(sys.stdin) if u.get('Username')=='alice']"

# Delete user
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/users/3
```

## Conclusion

Automatic user provisioning with Active Directory makes Portainer self-service - users log in with their corporate credentials and get appropriate access without any admin intervention. The key is establishing the AD group-to-Portainer-team mapping upfront, then configuring environment access per team. User lifecycle management (especially deprovisioning) still requires manual or scripted cleanup when users leave the organization.
