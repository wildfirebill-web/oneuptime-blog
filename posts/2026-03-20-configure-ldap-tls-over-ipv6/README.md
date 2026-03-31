# How to Configure LDAP TLS over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: LDAP, TLS, IPv6, StartTLS, OpenLDAP, Security, Certificate

Description: Configure TLS encryption for LDAP connections over IPv6 using LDAPS (port 636) and StartTLS, including certificate setup, client configuration, and verification.

---

Securing LDAP with TLS is essential in production environments. When combined with IPv6, you need certificates with appropriate Subject Alternative Names and correct IPv6 URI formatting for encrypted LDAP connections.

## Two TLS Methods for LDAP

1. **LDAPS** - Implicit TLS on port 636. The TLS handshake happens immediately on connection.
2. **StartTLS** - Plain LDAP on port 389 that upgrades to TLS using the STARTTLS extended operation.

Both work over IPv6 with the proper configuration.

## Generating Certificates with IPv6 SAN

Certificates must include SANs for the IPv6 address and hostname:

```bash
# Create OpenSSL config with IPv6 SAN

cat > /tmp/ldap-cert.cnf << 'EOF'
[req]
default_bits = 2048
prompt = no
distinguished_name = dn
x509_extensions = v3_req

[dn]
CN = ldap.example.com

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = ldap.example.com
DNS.2 = ldap
IP.1  = 2001:db8::1
IP.2  = ::1
EOF

# Generate private key
openssl genrsa -out /etc/ssl/private/ldap.key 2048

# Generate self-signed certificate (for testing)
openssl req -new -x509 \
  -key /etc/ssl/private/ldap.key \
  -out /etc/ssl/certs/ldap.crt \
  -days 365 \
  -config /tmp/ldap-cert.cnf

# For production: generate CSR and sign with CA
openssl req -new \
  -key /etc/ssl/private/ldap.key \
  -out /tmp/ldap.csr \
  -config /tmp/ldap-cert.cnf

# Verify the certificate has the IPv6 SAN
openssl x509 -in /etc/ssl/certs/ldap.crt -noout -text | \
  grep -A 5 "Subject Alternative"
```

## Configuring OpenLDAP for TLS over IPv6

```bash
# Configure TLS in slapd via ldapmodify
cat > /tmp/tls_config.ldif << 'EOF'
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/ldap.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/private/ldap.key
-
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/ca.crt
-
# Require TLS for all connections (optional, strict mode)
replace: olcSecurity
olcSecurity: tls=1
EOF

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/tls_config.ldif
```

Update slapd startup to enable LDAPS on IPv6:

```bash
# /etc/default/slapd
# Add ldaps:// to listen on IPv6
SLAPD_SERVICES="ldap:/// ldap://[::]/  ldaps:/// ldaps://[::]/  ldapi:///"
```

```bash
# Restart slapd
sudo systemctl restart slapd

# Verify LDAPS is listening on IPv6
ss -tlnp | grep :636
```

## Testing TLS LDAP Connections over IPv6

```bash
# Test LDAPS (implicit TLS) over IPv6
ldapsearch -H ldaps://[2001:db8::1]:636 \
  -x \
  -b "dc=example,dc=com" \
  "(objectClass=domain)" \
  -d 1  # Increase for more debug output

# Test StartTLS (explicit TLS) over IPv6
ldapsearch -H ldap://[2001:db8::1]:389 \
  -ZZ \
  -x \
  -b "dc=example,dc=com" \
  "(objectClass=domain)"

# Test TLS with certificate verification disabled (testing only)
LDAPTLS_REQCERT=never ldapsearch -H ldaps://[2001:db8::1]:636 \
  -x -b "dc=example,dc=com" "(objectClass=*)"

# Verify the certificate details in the TLS handshake
openssl s_client -connect '[2001:db8::1]':636 -showcerts < /dev/null
```

## Configuring LDAP Client TLS for IPv6

```bash
# /etc/ldap/ldap.conf (client configuration)

# LDAPS URI with IPv6 address
URI ldaps://[2001:db8::1]:636

# Base DN
BASE dc=example,dc=com

# CA certificate for verifying the server cert
TLS_CACERT /etc/ssl/certs/ca-certificates.crt

# Require valid certificate (demand = mandatory, try = optional, never = disable)
TLS_REQCERT demand

# Certificate for mutual TLS (client certificate, optional)
# TLS_CERT /etc/ssl/certs/client.crt
# TLS_KEY  /etc/ssl/private/client.key
```

## Configuring SSSD for LDAPS over IPv6

```ini
# /etc/sssd/sssd.conf

[domain/LDAP]
id_provider = ldap
auth_provider = ldap

# LDAPS URI with IPv6
ldap_uri = ldaps://[2001:db8::1]:636

# TLS settings
ldap_tls_cacert = /etc/ssl/certs/ca-certificates.crt
ldap_tls_reqcert = demand

# Optional: client certificate for mutual TLS
# ldap_tls_cert = /etc/ssl/certs/client.crt
# ldap_tls_key  = /etc/ssl/private/client.key
```

## Troubleshooting TLS LDAP over IPv6

```bash
# Check certificate SAN includes the address/hostname used in URI
openssl s_client -connect '[2001:db8::1]':636 < /dev/null | \
  openssl x509 -noout -text | grep "Subject Alternative"

# Common error: "TLS: hostname does not match CN in peer certificate"
# Fix: Ensure the certificate has IP.1 = 2001:db8::1 in SAN

# Check for cipher negotiation issues
openssl s_client -connect '[2001:db8::1]':636 \
  -tls1_2 -cipher 'HIGH:!aNULL' < /dev/null

# View detailed TLS debug output from slapd
sudo journalctl -u slapd | grep -i "tls\|ssl\|cert"
```

Configuring LDAP TLS over IPv6 requires certificates with IPv6-specific SANs and proper LDAP URI formatting, but provides the same strong encryption benefits as IPv4 TLS-secured LDAP connections.
