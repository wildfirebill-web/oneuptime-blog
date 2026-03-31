# How to Configure LDAP Group Search in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LDAP, Group, Authentication, Team, RBAC

Description: Configure LDAP group search in Portainer to automatically map directory groups to Portainer teams for role-based access control.

## Introduction

LDAP group search is what enables Portainer to automatically populate its teams based on LDAP/AD group membership. When configured, users logging in via LDAP are automatically added to corresponding Portainer teams, eliminating manual team management. This guide covers configuring group search for both OpenLDAP and Active Directory.

## How Group Search Works

1. User logs in with LDAP credentials
2. Portainer authenticates the user against the LDAP server
3. Portainer searches for groups containing the user
4. Portainer matches found groups to Portainer team names
5. User is added to matching teams automatically

## Configuring Group Search via UI

In Portainer → Settings → Authentication → LDAP:

### For OpenLDAP (groupOfNames)

```text
Group Base DN:            ou=groups,dc=example,dc=com
Group Membership Attr:    member
Group Filter:             (objectClass=groupOfNames)
```

### For OpenLDAP (posixGroup)

```text
Group Base DN:            ou=groups,dc=example,dc=com
Group Membership Attr:    memberUid
Group Filter:             (objectClass=posixGroup)
```

Note: `memberUid` contains plain usernames, while `member` contains full DNs. Portainer handles both.

### For Active Directory

```text
Group Base DN:            ou=groups,dc=corp,dc=example,dc=com
Group Membership Attr:    memberOf
Group Filter:             (objectClass=group)
```

For AD, the `memberOf` attribute is on the user object (not the group), so Portainer looks it up differently. Use:

```text
Group Attribute:          memberOf (AD user attribute)
```

## Configuring Group Search via API

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
          "Anonymous": false,
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

## Mapping LDAP Groups to Portainer Teams

For group-to-team mapping to work, you must:

1. **Create teams in Portainer** with names matching the LDAP group CNs
2. **Enable group search** in the LDAP configuration

For example, if you have:
- LDAP group: `cn=devops,ou=groups,dc=example,dc=com`
- Create Portainer team named: `devops`

When Alice (member of `devops` group) logs in, Portainer adds her to the `devops` team automatically.

### Case Sensitivity

Group names in Portainer must **exactly match** the LDAP group's `cn` attribute, including case. Check your LDAP group CNs:

```bash
ldapsearch -x \
  -H ldap://ldap.example.com:389 \
  -D "cn=portainer-bind,dc=example,dc=com" \
  -w bindpassword \
  -b "ou=groups,dc=example,dc=com" \
  "(objectClass=groupOfNames)" cn \
  | grep "^cn:"
```

## Testing Group Search

```bash
# Verify a user's group membership

ldapsearch -x \
  -H ldap://ldap.example.com:389 \
  -D "cn=portainer-bind,dc=example,dc=com" \
  -w bindpassword \
  -b "ou=groups,dc=example,dc=com" \
  "(member=uid=alice,ou=users,dc=example,dc=com)" cn

# For AD - find groups via memberOf on user object
ldapsearch -x \
  -H ldap://ad.corp.example.com:389 \
  -D "portainer-bind@corp.example.com" \
  -w bindpassword \
  -b "ou=users,dc=corp,dc=example,dc=com" \
  "(sAMAccountName=alice)" memberOf
```

## Auto-Populate Teams Feature (Portainer BE)

Portainer Business Edition supports automatic team creation from LDAP groups. Enable it in Settings → Authentication → LDAP → **Automatically populate teams from LDAP groups**.

With this enabled, Portainer creates teams for every LDAP group it finds, without requiring manual team creation.

## Conclusion

LDAP group search is the feature that makes group-based access control in Portainer automatic and scalable. Once configured, user-to-team assignments happen at login without any manual intervention. The key is naming your Portainer teams to exactly match your LDAP group CNs, and ensuring the bind account has read access to both user and group entries.
