# How to Configure IPsec IPv6 with Certificates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPsec, Certificate, PKI, strongSwan

Description: Learn how to configure IPv6 IPsec authentication using X.509 certificates with strongSwan, covering CA setup, certificate generation with IPv6 SANs, and connection configuration.

## Overview

Certificate-based authentication for IPv6 IPsec uses X.509 certificates to authenticate each peer. This is more scalable than PSK (one CA can authenticate thousands of gateways) and more secure (compromise of one certificate doesn't affect others). Certificates include IPv6 Subject Alternative Names (SANs) for proper identity binding.

## PKI Setup with strongSwan pki Tool

```bash
# ============================================================

# Step 1: Create Certificate Authority
# ============================================================
# Generate CA private key
pki --gen --type rsa --size 4096 --outform pem \
    > /etc/swanctl/private/ca.key.pem

# Self-sign CA certificate
pki --self --ca --lifetime 3650 \
    --in /etc/swanctl/private/ca.key.pem --type rsa \
    --dn "C=US, O=Corp VPN, CN=VPN Certificate Authority" \
    --outform pem \
    > /etc/swanctl/x509ca/ca.cert.pem

# ============================================================
# Step 2: Create Gateway Certificate for GW1
# ============================================================
# Generate GW1 private key
pki --gen --type rsa --size 2048 --outform pem \
    > /etc/swanctl/private/gw1.key.pem

# Create certificate request
pki --req --type rsa \
    --in /etc/swanctl/private/gw1.key.pem \
    --dn "C=US, O=Corp VPN, CN=gw1.example.com" \
    --san "gw1.example.com" \
    --san "2001:db8:gw1::1" \
    --outform pem \
    > /tmp/gw1.req.pem

# Sign with CA
pki --issue --lifetime 365 \
    --cacert /etc/swanctl/x509ca/ca.cert.pem \
    --cakey /etc/swanctl/private/ca.key.pem \
    --in /tmp/gw1.req.pem --type pkcs10 \
    --san "gw1.example.com" \
    --san "2001:db8:gw1::1" \
    --flag serverAuth --flag ikeIntermediate \
    --outform pem \
    > /etc/swanctl/x509/gw1.cert.pem

# Verify certificate
openssl x509 -in /etc/swanctl/x509/gw1.cert.pem -noout -text | grep -A5 'Subject Alternative'

# Set permissions
chmod 600 /etc/swanctl/private/*.pem
```

```bash
# ============================================================
# Step 3: Distribute Certificates to GW2
# ============================================================
# Copy to GW2 (securely):
scp /etc/swanctl/x509ca/ca.cert.pem root@gw2:/etc/swanctl/x509ca/
scp /tmp/gw2.cert.pem root@gw2:/etc/swanctl/x509/    # (generated separately on GW2)
# Note: Private keys NEVER leave the host they were generated on
```

## strongSwan Certificate Configuration

### GW1 /etc/swanctl/conf.d/vpn-cert.conf

```text
connections {
    gw1-to-gw2 {
        version = 2
        local_addrs  = 2001:db8:gw1::1
        remote_addrs = 2001:db8:gw2::1

        local {
            auth = pubkey
            certs = gw1.cert.pem     ! File in /etc/swanctl/x509/
            id = gw1.example.com     ! Must match certificate CN or SAN
        }
        remote {
            auth = pubkey
            id = gw2.example.com     ! Must match GW2's certificate CN or SAN
            # Alternatively, use ca= to accept any cert signed by our CA:
            # ca = "C=US, O=Corp VPN, CN=VPN Certificate Authority"
        }

        children {
            site-tunnel {
                local_ts  = 2001:db8:site1::/48
                remote_ts = 2001:db8:site2::/48
                mode = tunnel
                esp_proposals = aes256gcm128-prfsha256-ecp256
                start_action = start
            }
        }

        proposals = aes256-sha256-ecp256
    }
}
# No secrets{} block needed for certificate authentication
```

## Certificate Verification

```bash
# Load configuration and verify certificates load correctly
swanctl --load-all

# Check loaded credentials
swanctl --list-certs

# Expected output:
# List of X.509 End Entity Certificates:
#   subject:  "C=US, O=Corp VPN, CN=gw1.example.com"
#   issuer:   "C=US, O=Corp VPN, CN=VPN Certificate Authority"
#   validity:  not before Mar 20 00:00:00 2026, not after Mar 20 00:00:00 2027

# List CA certificates
swanctl --list-certs --ca

# Initiate tunnel
swanctl --initiate conn:gw1-to-gw2
```

## Certificate Revocation

```bash
# Generate Certificate Revocation List (CRL)
# When a certificate is compromised, revoke it:
openssl ca -revoke /etc/ssl/certs/compromised-gw.cert.pem \
    -keyfile /etc/swanctl/private/ca.key.pem \
    -cert /etc/swanctl/x509ca/ca.cert.pem

# Generate updated CRL
openssl ca -gencrl -out /etc/swanctl/crl/vpn-ca.crl.pem \
    -keyfile /etc/swanctl/private/ca.key.pem \
    -cert /etc/swanctl/x509ca/ca.cert.pem

# strongSwan automatically checks CRL if present in /etc/swanctl/crl/
```

## Certificate Rotation

```bash
# 30 days before expiry, generate new cert:
pki --gen --type rsa --size 2048 --outform pem > /etc/swanctl/private/gw1-new.key.pem
pki --req --type rsa --in /etc/swanctl/private/gw1-new.key.pem \
    --dn "CN=gw1.example.com" --san "gw1.example.com" --san "2001:db8:gw1::1" \
    --outform pem > /tmp/gw1-new.req.pem
pki --issue --lifetime 365 --cacert ca.cert.pem --cakey ca.key.pem \
    --in /tmp/gw1-new.req.pem --type pkcs10 \
    --outform pem > /etc/swanctl/x509/gw1-new.cert.pem

# Update swanctl.conf to reference new cert
# Reload without disruption
swanctl --load-creds
swanctl --initiate conn:gw1-to-gw2   # Rekeys with new cert
```

## Advantages Over PSK

| Feature | PSK | Certificate |
|---------|-----|-------------|
| Scalability | N PSKs for N peers | 1 CA for unlimited peers |
| Compromise impact | One PSK = one tunnel | Certificate revocation |
| Key rotation | Manual PSK update | Automated CA re-sign |
| Auditability | Difficult | Certificate logs |
| Perfect forward secrecy | IKE provides PFS | IKE provides PFS |

## Summary

Certificate-based IPv6 IPsec uses strongSwan's `pki` tool to create a CA, generate per-gateway certificates with IPv6 Subject Alternative Names, and configure `auth = pubkey` with `certs =` in swanctl.conf. Distribute only certificates and the CA cert - never private keys. strongSwan validates certificates against the CA in `/etc/swanctl/x509ca/` and checks CRLs if present. For large deployments, integrate with an enterprise CA (Active Directory, Vault) and use SCEP or EST for automated certificate enrollment.
