# How to Configure AD Group-Based Access in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Active Directory, RBAC, Team, Access Control

Description: Set up role-based access control in Portainer using Active Directory security groups to map AD groups to Portainer teams and roles.

---

Group-based access control in Portainer means AD group membership determines what environments and resources users can access. This scales access management across hundreds of users without individual configuration.

## Design Your Access Model

Before configuration, plan the mapping:

```text
AD Group                    → Portainer Team         → Access
---------------------------------------------------------------------------
GRP-Portainer-Admins       → portainer-admins       → All environments (Admin)
GRP-Portainer-DevOps       → devops-team            → Production (Operator)
GRP-Portainer-Developers   → developers             → Development (Standard)
GRP-Portainer-ReadOnly     → read-only-users        → All environments (Read)
```

## Step 1: Create AD Security Groups

```powershell
# Run on a domain controller

New-ADGroup -Name "GRP-Portainer-Admins" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "GRP-Portainer-DevOps" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "GRP-Portainer-Developers" -GroupScope Global -GroupCategory Security
New-ADGroup -Name "GRP-Portainer-ReadOnly" -GroupScope Global -GroupCategory Security

# Add users to groups
Add-ADGroupMember -Identity "GRP-Portainer-Admins" -Members "jsmith"
Add-ADGroupMember -Identity "GRP-Portainer-Developers" -Members "jdoe", "bwilson"
```

## Step 2: Create Matching Teams in Portainer

```bash
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create teams matching AD group names (or a mapping)
for team in "portainer-admins" "devops-team" "developers" "read-only-users"; do
  RESULT=$(curl -s -X POST \
    https://localhost:9443/api/teams \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"Name\": \"$team\"}" \
    --insecure)
  echo "Team created: $team -> $(echo $RESULT | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d.get("Id","error"))')"
done
```

## Step 3: Configure AD with Group Sync

```bash
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "AuthenticationMethod": 2,
    "LDAPSettings": {
      "ReaderDN": "CN=portainer-svc,OU=Service Accounts,DC=corp,DC=example,DC=com",
      "Password": "ServicePassword!",
      "URLs": ["ldaps://dc01.corp.example.com:636"],
      "TLSConfig": {"TLS": true, "TLSSkipVerify": false},
      "SearchSettings": [{
        "BaseDN": "DC=corp,DC=example,DC=com",
        "Filter": "(&(objectClass=user)(objectCategory=person))",
        "UserNameAttribute": "sAMAccountName"
      }],
      "GroupSearchSettings": [{
        "GroupBaseDN": "OU=Portainer Groups,DC=corp,DC=example,DC=com",
        "GroupFilter": "(objectClass=group)",
        "UserAttribute": "member",
        "GroupAttribute": "cn"
      }],
      "AutoCreateUsers": true
    }
  }' \
  --insecure
```

## Step 4: Assign Teams to Environments

After teams are created and users are synced, assign environment access:

```bash
# Get the list of environments and teams
ENVS=$(curl -s https://localhost:9443/api/endpoints -H "Authorization: Bearer $TOKEN" --insecure)
TEAMS=$(curl -s https://localhost:9443/api/teams -H "Authorization: Bearer $TOKEN" --insecure)

echo "Environments:"
echo "$ENVS" | python3 -c "import sys,json; [print(f'  {e[\"Id\"]}: {e[\"Name\"]}') for e in json.load(sys.stdin)]"

echo "Teams:"
echo "$TEAMS" | python3 -c "import sys,json; [print(f'  {t[\"Id\"]}: {t[\"Name\"]}') for t in json.load(sys.stdin)]"
```

Then assign in the Portainer UI: **Environments > [Select Environment] > Access** - assign teams with appropriate roles.

## Verify Group-Based Access

Log in as a user who is a member of `GRP-Portainer-Developers` and verify they can only see the development environment with Standard User permissions.

---

*Monitor your team's container infrastructure with [OneUptime](https://oneuptime.com) for comprehensive observability.*
