# How to Auto-Populate Teams from LDAP Groups in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LDAP, Teams, Authentication, Business Edition

Description: Configure Portainer Business Edition to automatically create and populate teams based on LDAP group membership.

---

Portainer Business Edition can map LDAP groups to Portainer teams, automatically adding users to the correct teams when they log in. This eliminates manual team management for organizations using LDAP or Active Directory.

## How LDAP Team Sync Works

When a user logs in via LDAP:
1. Portainer authenticates the user against LDAP
2. Portainer queries the user's group memberships
3. LDAP groups are mapped to Portainer teams
4. The user is added to the corresponding teams automatically

## Configure LDAP with Group Sync via UI

1. Navigate to **Settings > Authentication**
2. Select **LDAP** authentication
3. Enable **Auto-populate team membership**
4. Configure the **Group search settings**:
   - **Group Base DN**: `ou=groups,dc=example,dc=com`
   - **Group Filter**: `(objectClass=groupOfNames)` or `(objectClass=group)` for AD
   - **Group Attribute for Users**: `member` (or `memberOf` depending on schema)
   - **Group Name Attribute**: `cn`

## Configure via API

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure LDAP with group auto-population

curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 2,
    "LDAPSettings": {
      "AnonymousMode": false,
      "ReaderDN": "cn=portainer,dc=example,dc=com",
      "Password": "ldap-password",
      "URLs": ["ldaps://ldap.example.com:636"],
      "TLSConfig": {"TLS": true, "TLSSkipVerify": false},
      "SearchSettings": [{
        "BaseDN": "ou=users,dc=example,dc=com",
        "Filter": "(objectClass=inetOrgPerson)",
        "UserNameAttribute": "uid"
      }],
      "GroupSearchSettings": [{
        "GroupBaseDN": "ou=groups,dc=example,dc=com",
        "GroupFilter": "(objectClass=groupOfNames)",
        "UserAttribute": "member",
        "GroupAttribute": "cn"
      }],
      "AutoCreateUsers": true
    }
  }' \
  --insecure
```

## Create Portainer Teams to Match LDAP Groups

Portainer teams must exist before LDAP group sync can map users to them:

```bash
# Create teams via API to match your LDAP group names
# Team names must match the LDAP group cn attribute values

for team in "developers" "operations" "qa-team" "devops"; do
  curl -s -X POST \
    https://localhost:9443/api/teams \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"Name\": \"$team\"}" \
    --insecure
  echo "Created team: $team"
done
```

## Active Directory Group Sync

For Active Directory, the memberOf attribute is typically used:

```json
{
  "GroupSearchSettings": [{
    "GroupBaseDN": "ou=groups,dc=corp,dc=example,dc=com",
    "GroupFilter": "(objectClass=group)",
    "UserAttribute": "memberOf",
    "GroupAttribute": "cn"
  }]
}
```

## Verify Group Sync

After a user logs in via LDAP, verify they've been added to the correct team:

```bash
# List teams and their members
curl -s https://localhost:9443/api/teams \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -m json.tool

# Check a specific user's team membership
curl -s https://localhost:9443/api/users \
  -H "Authorization: Bearer $TOKEN" \
  --insecure | python3 -c "
import sys, json
users = json.load(sys.stdin)
for u in users:
    print(f'User: {u[\"Username\"]}, Teams: {u.get(\"TeamIDs\", [])}')
"
```

---

*Scale your team access management and monitor infrastructure with [OneUptime](https://oneuptime.com).*
