# How to Set Up LDAP with StartTLS Encryption in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LDAP, StartTLS, Security, Encryption, TLS

Description: Configure Portainer to use StartTLS encryption when connecting to your LDAP server on port 389 for secure credential transmission.

## Introduction

StartTLS upgrades a plain LDAP connection (port 389) to an encrypted TLS connection after the initial handshake. It's a middle ground between unencrypted LDAP and full LDAPS (port 636) — you use the standard LDAP port but get encryption. This guide configures Portainer to use StartTLS with your LDAP server.

## StartTLS vs LDAPS

| Feature | LDAP (plain) | StartTLS | LDAPS |
|---------|-------------|----------|-------|
| Port | 389 | 389 | 636 |
| Encryption | None | Upgraded TLS | Full TLS |
| Certificate | Not required | Required | Required |
| Compatibility | All servers | Modern servers | All servers |

StartTLS is preferred when your environment uses port 389 exclusively (firewall rules) but you need encryption.

## Prerequisites

- LDAP server configured to support StartTLS
- CA certificate (or self-signed cert) for the LDAP server's TLS certificate
- Portainer running

## Step 1: Verify Your LDAP Server Supports StartTLS

```bash
# Test StartTLS connectivity
ldapsearch -x \
  -H ldap://ldap.example.com:389 \
  -Z \
  -D "cn=portainer-bind,dc=example,dc=com" \
  -w bindpassword \
  -b "dc=example,dc=com" \
  "(objectClass=*)" dn

# The -Z flag forces StartTLS
# -ZZ would require StartTLS and fail if not available
ldapsearch -x -H ldap://ldap.example.com:389 -ZZ \
  -D "cn=portainer-bind,dc=example,dc=com" \
  -w bindpassword \
  -b "dc=example,dc=com" \
  -s base "(objectClass=*)" supportedExtension
```

## Step 2: Obtain the LDAP Server's CA Certificate

```bash
# Method 1: Retrieve from the LDAP server
openssl s_client -connect ldap.example.com:389 -starttls ldap < /dev/null 2>/dev/null \
  | openssl x509 -noout -text

# Save the certificate
openssl s_client -connect ldap.example.com:389 -starttls ldap < /dev/null 2>/dev/null \
  | openssl x509 > ldap-ca.pem

# Method 2: Get from your certificate authority
# Download the CA certificate from your internal PKI

# Verify the certificate
openssl x509 -in ldap-ca.pem -text -noout | grep -E "Subject:|Issuer:|Not After:"
```

## Step 3: Configure Portainer for StartTLS

In Settings → Authentication → LDAP:

```
Server:              ldap.example.com:389
StartTLS:            Enabled (toggle ON)
Skip TLS Verify:     Off (for production - verify the certificate)
TLS CA Certificate:  [paste the PEM certificate content]
```

**PEM Format Example:**
```
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAJC1HiIAZAiIMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
... (certificate content) ...
-----END CERTIFICATE-----
```

## Step 4: Configure via API

```bash
# Read the CA certificate
CA_CERT=$(cat ldap-ca.pem)

TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/settings \
  -d "{
    \"AuthenticationMethod\": 2,
    \"ldapsettings\": {
      \"Servers\": [
        {
          \"Host\": \"ldap.example.com\",
          \"Port\": 389,
          \"UseTLS\": false,
          \"StartTLS\": true,
          \"SkipVerify\": false,
          \"TLSCACert\": $(python3 -c \"import sys; print(__import__('json').dumps(open('ldap-ca.pem').read()))\"),
          \"Anonymous\": false,
          \"ReaderDN\": \"cn=portainer-bind,dc=example,dc=com\",
          \"Password\": \"bindpassword\"
        }
      ],
      \"SearchSettings\": [
        {
          \"BaseDN\": \"ou=users,dc=example,dc=com\",
          \"Username\": \"uid\",
          \"Filter\": \"(objectClass=inetOrgPerson)\"
        }
      ]
    }
  }"
```

## Step 5: Testing the StartTLS Connection

After configuring, use the built-in test in Portainer's LDAP settings:

```bash
# Test from the command line
ldapsearch -x \
  -H ldap://ldap.example.com:389 \
  -ZZ \
  -D "cn=portainer-bind,dc=example,dc=com" \
  -w bindpassword \
  -b "ou=users,dc=example,dc=com" \
  "(uid=testuser)" uid cn

# Expected: returns user entry if successful
# Error: "ldap_start_tls: Connect error (-11)" means StartTLS not supported
```

## Troubleshooting

**"TLS handshake failure"**: The CA certificate is wrong or incomplete. Ensure you have the full certificate chain.

**"Certificate verify failed"**: The LDAP server's hostname doesn't match the certificate's CN/SAN. Check the certificate's Subject Alternative Names.

**"StartTLS not supported"**: The LDAP server doesn't have TLS configured. Check your LDAP server's TLS configuration.

To temporarily diagnose, enable **Skip TLS Verification** to confirm it's a certificate issue, then fix the certificate and disable skip.

## Conclusion

StartTLS provides encryption for LDAP traffic without requiring port 636 or a separate LDAPS listener. Once configured with the correct CA certificate, Portainer seamlessly upgrades all LDAP connections to TLS. For new installations, LDAPS (port 636) is generally simpler, but StartTLS is the right choice when your network infrastructure mandates port 389.
