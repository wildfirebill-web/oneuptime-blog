# How to Configure Active Directory Authentication in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Active Directory, LDAP, Authentication, Window, Enterprise

Description: Configure Portainer to authenticate users against Microsoft Active Directory using LDAP with AD-specific settings and service accounts.

## Introduction

Active Directory (AD) is Microsoft's directory service used in most enterprise Windows environments. Portainer connects to AD using the LDAP protocol, but AD has specific conventions that differ from OpenLDAP - different attribute names, DN formats, and filter syntax. This guide covers AD-specific configuration.

## Prerequisites

- Active Directory domain controller accessible from Portainer
- Service account in AD with read access (Domain Users is usually sufficient)
- Domain name and DC server address

## Step 1: Create a Service Account in Active Directory

Create a dedicated low-privilege service account for Portainer:

```powershell
# Run on a Domain Controller or admin workstation

New-ADUser `
  -Name "portainer-svc" `
  -SamAccountName "portainer-svc" `
  -UserPrincipalName "portainer-svc@corp.example.com" `
  -AccountPassword (ConvertTo-SecureString "ServiceP@ssword123" -AsPlainText -Force) `
  -PasswordNeverExpires $true `
  -Enabled $true `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com"

# The account needs read access to Users and Groups OUs
# Domain Users group membership is typically sufficient
```

## Step 2: Gather AD Connection Information

```powershell
# On a domain-joined Windows machine, find DC server
nslookup -type=SRV _ldap._tcp.corp.example.com

# Or use:
(Get-ADDomainController -Discover).HostName

# Get Domain DN
(Get-ADDomain).DistinguishedName
# Example: DC=corp,DC=example,DC=com

# Find users OU
Get-ADOrganizationalUnit -Filter * | Select Name,DistinguishedName
```

## Step 3: Configure Portainer LDAP for Active Directory

In Settings → Authentication → LDAP:

```text
Server:             dc01.corp.example.com:389
                    (or use LDAPS: dc01.corp.example.com:636)

Service Account:
  Using UPN:        portainer-svc@corp.example.com
  OR using DN:      CN=portainer-svc,OU=Service Accounts,DC=corp,DC=example,DC=com
Password:           ServiceP@ssword123

User Base DN:       DC=corp,DC=example,DC=com
Username Attribute: sAMAccountName
User Filter:        (&(objectClass=user)(objectCategory=person))

Group Base DN:      DC=corp,DC=example,DC=com
Group Attribute:    memberOf
Group Filter:       (objectClass=group)
```

## Step 4: API Configuration for Active Directory

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
          "UseTLS": false,
          "StartTLS": false,
          "SkipVerify": false,
          "Anonymous": false,
          "ReaderDN": "portainer-svc@corp.example.com",
          "Password": "ServiceP@ssword123"
        }
      ],
      "SearchSettings": [
        {
          "BaseDN": "DC=corp,DC=example,DC=com",
          "Username": "sAMAccountName",
          "Filter": "(&(objectClass=user)(objectCategory=person))"
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

## Key Differences: AD vs OpenLDAP

| Setting | Active Directory | OpenLDAP |
|---------|----------------|---------|
| Username Attribute | `sAMAccountName` | `uid` |
| User Filter | `(&(objectClass=user)(objectCategory=person))` | `(objectClass=inetOrgPerson)` |
| Group Filter | `(objectClass=group)` | `(objectClass=groupOfNames)` |
| Group Membership | `memberOf` on user object | `member` on group object |
| Bind Account Format | `user@domain.com` or CN=... | `cn=user,dc=...` |
| Disabled Accounts | Filter with `userAccountControl` | No standard method |

## Filtering Out Disabled Accounts

```text
User Filter: (&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
```

The `userAccountControl` bitmask filter excludes disabled AD accounts.

## Testing AD Connectivity

```bash
# Test from Linux with ldapsearch
ldapsearch -x \
  -H ldap://dc01.corp.example.com:389 \
  -D "portainer-svc@corp.example.com" \
  -w "ServiceP@ssword123" \
  -b "DC=corp,DC=example,DC=com" \
  "(sAMAccountName=alice)" sAMAccountName displayName mail memberOf
```

## Conclusion

Active Directory authentication in Portainer is LDAP-based but requires AD-specific attribute names and filter syntax. The key differences are using `sAMAccountName` (or UPN) as the username attribute, `objectClass=user` with `objectCategory=person` as the user filter, and using the `memberOf` attribute for group membership. Once configured correctly, AD authentication works identically to OpenLDAP from the user's perspective.
