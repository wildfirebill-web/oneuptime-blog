# How to Configure AD Group-Based Access in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Active Directory, Groups, RBAC, Access Control

Description: Map Active Directory security groups to Portainer teams to control environment access based on AD group membership.

## Introduction

Active Directory group-based access in Portainer enables you to control who can access which environments based on existing AD security group membership. This integrates Portainer's RBAC with your AD group structure, so access changes in AD are reflected in Portainer at the user's next login.

## Architecture Overview

```text
AD Security Group: CN=Portainer-DevOps,OU=Groups,DC=corp,DC=example,DC=com
        |
        v
Portainer Team: "Portainer-DevOps"
        |
        v
Environment Access: Production Environment (Standard User role)
```

## Step 1: Create AD Security Groups

```powershell
# Create groups for different access levels

New-ADGroup -Name "Portainer-DevOps" `
  -SamAccountName "Portainer-DevOps" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Groups,DC=corp,DC=example,DC=com" `
  -Description "Portainer DevOps team - full environment access"

New-ADGroup -Name "Portainer-QA" `
  -SamAccountName "Portainer-QA" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Groups,DC=corp,DC=example,DC=com" `
  -Description "Portainer QA team - staging access"

New-ADGroup -Name "Portainer-ReadOnly" `
  -SamAccountName "Portainer-ReadOnly" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Groups,DC=corp,DC=example,DC=com" `
  -Description "Portainer read-only access"

# Add members to groups
Add-ADGroupMember -Identity "Portainer-DevOps" -Members alice,bob
Add-ADGroupMember -Identity "Portainer-QA" -Members charlie,diana
Add-ADGroupMember -Identity "Portainer-ReadOnly" -Members eve
```

## Step 2: Configure Portainer LDAP with Group Search

In Settings → Authentication → LDAP:

```text
Group Base DN:            OU=Groups,DC=corp,DC=example,DC=com
Group Membership Attr:    memberOf
Group Filter:             (objectClass=group)
```

For AD, `memberOf` is an attribute on the user object that lists all groups the user belongs to. Portainer reads this attribute to determine team membership.

## Step 3: Create Matching Teams in Portainer

Create Portainer teams with names matching the AD group CNs:

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Create teams matching AD group names
for team in "Portainer-DevOps" "Portainer-QA" "Portainer-ReadOnly"; do
  curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    https://portainer.example.com/api/teams \
    -d "{\"name\": \"${team}\"}"
  echo "Created team: $team"
done
```

## Step 4: Assign Teams to Environments

```bash
# Get list of environments
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "import sys,json; [print(f'ID={e[\"Id\"]} Name={e[\"Name\"]}') for e in json.load(sys.stdin)]"

# Get team IDs
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/teams \
  | python3 -c "import sys,json; [print(f'ID={t[\"Id\"]} Name={t[\"Name\"]}') for t in json.load(sys.stdin)]"

# Assign team access to environment
# Environment ID: 1, Team Portainer-DevOps ID: 1, Role: 2 (Standard User)
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/teamaccesspolicies \
  -d '{
    "1": {"RoleId": 2}
  }'
```

Role IDs:
- `1` = Read-only (Helpdesk)
- `2` = Standard User
- `3` = Operator (Kubernetes)

## Step 5: Enable Automatic Team Sync

With LDAP configured and groups set up:

1. Log out of Portainer
2. Log back in with an AD account that's a member of `Portainer-DevOps`
3. Portainer should automatically add the user to the `Portainer-DevOps` team

## Handling Nested AD Groups

If your users belong to nested AD groups, use the recursive memberOf LDAP_MATCHING_RULE_IN_CHAIN OID:

```text
Group Filter: (memberOf:1.2.840.113556.1.4.1941:=CN=Portainer-DevOps,OU=Groups,DC=corp,DC=example,DC=com)
```

This matches direct and indirect (nested) group membership.

## Conclusion

AD group-based access control in Portainer bridges your existing AD group structure with container infrastructure access. By naming Portainer teams to match AD group CNs, you create a direct mapping that's maintained automatically. Access changes in AD (adding/removing group members) take effect the next time the user logs in to Portainer, with no manual Portainer changes required.
