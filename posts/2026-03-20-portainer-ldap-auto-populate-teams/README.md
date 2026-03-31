# How to Auto-Populate Teams from LDAP Groups in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LDAP, Team, Automation, Business Edition, RBAC

Description: Configure Portainer Business Edition to automatically create and populate teams based on LDAP group membership at user login.

## Introduction

Manually creating Portainer teams and mapping them to LDAP groups works but doesn't scale. Portainer Business Edition provides automatic team population from LDAP groups - when enabled, Portainer creates teams for LDAP groups and adds users to those teams automatically at login. This eliminates all manual team management.

## Prerequisites

- Portainer Business Edition
- LDAP authentication configured and working
- LDAP group search configured
- Admin access to Portainer

## How Auto-Population Works

With auto-population enabled:

1. User logs in with LDAP credentials
2. Portainer authenticates the user
3. Portainer queries LDAP for the user's group memberships
4. For each LDAP group:
   - If a matching Portainer team exists: user is added to it
   - If no matching team exists: Portainer creates the team and adds the user
5. User is removed from teams whose corresponding LDAP groups they no longer belong to

This provides full lifecycle management: teams are created, populated, and cleaned up automatically.

## Step 1: Enable Group Search

First, ensure group search is configured:

```text
Group Base DN:     ou=groups,dc=example,dc=com
Group Attribute:   member
Group Filter:      (objectClass=groupOfNames)
```

## Step 2: Enable Automatic Team Population

In Settings → Authentication → LDAP:

1. Scroll to the **Teams** section
2. Enable **Automatic team membership synchronization**
3. Save settings

## Step 3: Configure via API

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
          "Host": "ldap.example.com",
          "Port": 389,
          "ReaderDN": "cn=portainer-bind,dc=example,dc=com",
          "Password": "bindpassword"
        }
      ],
      "SearchSettings": [
        {
          "BaseDN": "ou=users,dc=example,dc=com",
          "Username": "uid",
          "Filter": "(objectClass=inetOrgPerson)"
        }
      ],
      "GroupSearchSettings": [
        {
          "GroupBaseDN": "ou=groups,dc=example,dc=com",
          "GroupAttribute": "member",
          "GroupFilter": "(objectClass=groupOfNames)"
        }
      ],
      "AutoCreateUsers": true
    }
  }'
```

## LDAP Group Structure Example

```ldif
# DevOps team group

dn: cn=devops,ou=groups,dc=example,dc=com
objectClass: groupOfNames
cn: devops
member: uid=alice,ou=users,dc=example,dc=com
member: uid=bob,ou=users,dc=example,dc=com

# QA team group
dn: cn=qa-team,ou=groups,dc=example,dc=com
objectClass: groupOfNames
cn: qa-team
member: uid=charlie,ou=users,dc=example,dc=com
member: uid=diana,ou=users,dc=example,dc=com

# Admin group - maps to Portainer admins
dn: cn=portainer-admins,ou=groups,dc=example,dc=com
objectClass: groupOfNames
cn: portainer-admins
member: uid=admin-user,ou=users,dc=example,dc=com
```

After configuration, when Alice logs in, Portainer:
1. Creates team `devops` (if it doesn't exist)
2. Adds Alice to `devops` team
3. Grants Alice access to environments assigned to `devops`

## Assigning Environment Access to Auto-Created Teams

Auto-created teams don't have environment access by default. You must assign it:

1. Go to **Environments** → click the environment
2. Click **Manage access**
3. Add the team (e.g., `devops`) with the appropriate role

Or via API:
```bash
ENDPOINT_ID=1
TEAM_ID=2   # The auto-created team's ID

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}/teamaccesspolicies" \
  -d '{
    "2": {
      "RoleId": 2
    }
  }'
```

## Auto-Admin Assignment from LDAP

To automatically make members of a specific LDAP group Portainer administrators:

1. Create an LDAP group called `portainer-admins`
2. In Portainer LDAP settings, configure the admin group:

```text
Admin Group:  portainer-admins
```

Members of this group get admin privileges automatically at login.

## Conclusion

Automatic team population from LDAP groups is a powerful feature of Portainer Business Edition that eliminates all manual team management. Users' Portainer access automatically reflects their directory group membership, and changes in the directory are reflected immediately at the user's next login. The remaining manual step is assigning environment access to the auto-created teams.
