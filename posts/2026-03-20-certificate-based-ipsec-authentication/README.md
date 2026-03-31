# How to Set Up Certificate-Based IPsec Authentication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPsec, PKI, Certificate, strongSwan, IKEv2, Linux

Description: Configure IPsec VPN authentication using X.509 certificates with strongSwan, providing more scalable and secure authentication than pre-shared keys.

Certificate-based IPsec authentication eliminates shared secrets and enables scalable deployment. Each gateway or client has a unique certificate, and revocation is possible without affecting other users.

## Step 1: Create a Certificate Authority

```bash
# Generate the CA private key

ipsec pki --gen --type rsa --size 4096 --outform pem > /etc/ipsec.d/private/ca.key.pem
chmod 600 /etc/ipsec.d/private/ca.key.pem

# Create the self-signed CA certificate
ipsec pki --self --ca --lifetime 3650 \
  --in /etc/ipsec.d/private/ca.key.pem \
  --type rsa \
  --dn "C=US, O=My Org, CN=IPsec Root CA" \
  --outform pem > /etc/ipsec.d/cacerts/ca.cert.pem
```

## Step 2: Generate Gateway Certificates

For each gateway, create a key pair and certificate signed by the CA:

```bash
# Gateway A certificate
ipsec pki --gen --type rsa --size 2048 --outform pem > /etc/ipsec.d/private/gateway-a.key.pem

ipsec pki --pub --in /etc/ipsec.d/private/gateway-a.key.pem --type rsa | \
  ipsec pki --issue --lifetime 1825 \
  --cacert /etc/ipsec.d/cacerts/ca.cert.pem \
  --cakey /etc/ipsec.d/private/ca.key.pem \
  --dn "C=US, O=My Org, CN=Gateway A" \
  --san "gateway-a.example.com" \
  --flag serverAuth \
  --outform pem > /etc/ipsec.d/certs/gateway-a.cert.pem

# Verify the certificate
ipsec pki --print --in /etc/ipsec.d/certs/gateway-a.cert.pem
```

## Step 3: Configure ipsec.conf for Certificate Auth

```conf
# /etc/ipsec.conf on Gateway A

conn cert-tunnel
    keyexchange=ikev2
    auto=start
    type=tunnel

    left=1.2.3.4
    leftid="C=US, O=My Org, CN=Gateway A"
    leftcert=gateway-a.cert.pem      # Filename in /etc/ipsec.d/certs/
    leftsubnet=10.1.0.0/24
    leftauth=pubkey                  # Certificate authentication

    right=5.6.7.8
    rightid="C=US, O=My Org, CN=Gateway B"
    rightsubnet=10.2.0.0/24
    rightauth=pubkey

    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
```

## Step 4: Configure ipsec.secrets for Certificate

```conf
# /etc/ipsec.secrets
# Reference the private key file for this gateway's certificate
: RSA gateway-a.key.pem
```

## Step 5: Copy Certificates for Gateway B

```bash
# Transfer to Gateway B (use secure channel)
scp /etc/ipsec.d/cacerts/ca.cert.pem admin@gateway-b:/etc/ipsec.d/cacerts/
scp /etc/ipsec.d/certs/gateway-b.cert.pem admin@gateway-b:/etc/ipsec.d/certs/
scp /etc/ipsec.d/private/gateway-b.key.pem admin@gateway-b:/etc/ipsec.d/private/
```

## Verifying Certificate-Based Authentication

```bash
# Start strongSwan and bring up the tunnel
sudo systemctl restart strongswan
sudo ipsec up cert-tunnel

# Verify authentication succeeded
sudo journalctl -u strongswan | grep -i "cert\|pubkey\|authenticated"
# Expected: "authentication of ... with RSA signature successful"

# Check tunnel status
sudo ipsec statusall | grep "ESTABLISHED"
```

## Certificate Revocation with CRL

```bash
# If a gateway's certificate is compromised, revoke it
# On the CA host:

# Revoke a certificate
ipsec pki --gen-crl \
  --cacert /etc/ipsec.d/cacerts/ca.cert.pem \
  --cakey /etc/ipsec.d/private/ca.key.pem \
  --cert /etc/ipsec.d/certs/compromised-gateway.cert.pem \
  > /etc/ipsec.d/crls/revoked.crl.pem

# Distribute the CRL to all gateways
# Gateways load CRLs from /etc/ipsec.d/crls/ automatically
```

Certificate-based IPsec authentication is the production standard for deployments with multiple gateways or clients, providing per-device accountability and immediate revocation capability.
