# How to Set Up LDAP with TLS Encryption in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LDAP, TLS, Security, Encryption

Description: Configure Portainer to connect to your LDAP server using LDAPS (LDAP over TLS) on port 636 for end-to-end encrypted authentication.

## Introduction

LDAPS (LDAP over SSL/TLS) uses port 636 and wraps the entire LDAP connection in TLS from the start - unlike StartTLS which upgrades a plain connection. LDAPS is simpler to configure and is the recommended encryption method for new deployments. This guide covers enabling LDAPS in Portainer.

## Prerequisites

- LDAP server with LDAPS enabled on port 636
- CA certificate for the LDAP server
- Port 636 accessible from Portainer

## Step 1: Verify LDAPS Availability

```bash
# Test direct LDAPS connection

openssl s_client -connect ldap.example.com:636 < /dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -E "Subject:|Not After:"

# Test with ldapsearch
ldapsearch -x \
  -H ldaps://ldap.example.com:636 \
  -D "cn=portainer-bind,dc=example,dc=com" \
  -w bindpassword \
  -b "dc=example,dc=com" \
  -s base "(objectClass=*)"
```

## Step 2: Export the Server Certificate

```bash
# Save the LDAP server's CA certificate
openssl s_client -connect ldap.example.com:636 < /dev/null 2>/dev/null \
  | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > ldap-server.pem

# For Active Directory, get the CA cert from AD CS
# The CA cert is usually available at:
# http://your-ca-server/certsrv/certcarc.asp

# View certificate details
openssl x509 -in ldap-server.pem -noout -text | grep -E "DNS:|IP:"
```

## Step 3: Configure Portainer for LDAPS

In Settings → Authentication → LDAP:

```text
Server:             ldap.example.com:636
Use TLS:            Enabled (LDAPS)
StartTLS:           Disabled
Skip TLS Verify:    Off (use cert verification in production)
TLS CA Certificate: [paste PEM certificate here]

Reader DN:          cn=portainer-bind,dc=example,dc=com
Reader Password:    bindpassword
```

## Step 4: API Configuration

```bash
# Read CA cert
CA_PEM=$(cat ldap-server.pem)

TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure LDAPS
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d "$(python3 -c "
import json
ca_cert = open('ldap-server.pem').read()
config = {
  'AuthenticationMethod': 2,
  'ldapsettings': {
    'Servers': [
      {
        'Host': 'ldap.example.com',
        'Port': 636,
        'UseTLS': True,
        'StartTLS': False,
        'SkipVerify': False,
        'TLSCACert': ca_cert,
        'Anonymous': False,
        'ReaderDN': 'cn=portainer-bind,dc=example,dc=com',
        'Password': 'bindpassword'
      }
    ],
    'SearchSettings': [
      {
        'BaseDN': 'ou=users,dc=example,dc=com',
        'Username': 'uid',
        'Filter': '(objectClass=inetOrgPerson)'
      }
    ],
    'GroupSearchSettings': [
      {
        'GroupBaseDN': 'ou=groups,dc=example,dc=com',
        'GroupAttribute': 'member',
        'GroupFilter': '(objectClass=groupOfNames)'
      }
    ],
    'AutoCreateUsers': True
  }
}
print(json.dumps(config))
")"
```

## Active Directory LDAPS Configuration

For Active Directory with LDAPS:

```text
Server:             dc01.corp.example.com:636
Use TLS:            Enabled
Skip TLS Verify:    Off (unless using self-signed AD cert)
TLS CA Certificate: [Root CA certificate from AD CS]

Reader DN:          CN=portainer-bind,OU=Service Accounts,DC=corp,DC=example,DC=com
Reader Password:    [service account password]

User Base DN:       DC=corp,DC=example,DC=com
Username Attribute: sAMAccountName
User Filter:        (&(objectClass=user)(objectCategory=person))
```

## Testing LDAPS After Configuration

Use Portainer's built-in test in the LDAP settings page. For manual testing:

```bash
# Test from the Portainer container
docker exec portainer \
  wget -q -O- --no-check-certificate \
  "https://ldap.example.com:636" 2>&1 | head -5

# Or use openssl from Portainer host
openssl s_client -connect ldap.example.com:636 \
  -CAfile ldap-server.pem < /dev/null
# Look for: "Verify return code: 0 (ok)"
```

## Handling Self-Signed Certificates

In development or small environments with self-signed LDAP certs:

```text
Skip TLS Verify: On
```

This disables certificate verification. Only use in non-production environments where you control the LDAP server and network.

For production with self-signed certs, properly import the self-signed certificate as the CA cert instead of skipping verification.

## Firewall Configuration

Ensure port 636 is open from Portainer to the LDAP server:

```bash
# Test port connectivity
nc -zv ldap.example.com 636

# Or from Docker
docker run --rm alpine nc -zv ldap.example.com 636
```

## Conclusion

LDAPS provides the simplest and most reliable TLS configuration for LDAP authentication in Portainer. By using port 636, the entire connection is encrypted from the first byte, which is more secure than StartTLS. Always use certificate verification in production by providing the CA certificate, and test connectivity before configuring Portainer to save troubleshooting time.
