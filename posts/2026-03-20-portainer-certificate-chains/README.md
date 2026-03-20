# How to Configure Certificate Chains in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, SSL, TLS, Certificate-chain, PKI, Intermediate-ca

Description: A guide to properly configuring certificate chains (including intermediate CAs) in Portainer to avoid trust errors.

## Overview

Enterprise PKI environments commonly use certificate chains: a root CA signs an intermediate CA, which signs server certificates. Browsers and API clients need the complete chain to verify trust. If only the server certificate is provided (without the intermediate), clients that don't have the intermediate in their trust store will fail verification. This guide covers assembling and configuring proper certificate chains in Portainer.

## Prerequisites

- Portainer running with SSL configured
- Certificate files from your CA (server cert, intermediate cert(s), root cert)
- OpenSSL for verification

## Understanding Certificate Chains

```text
Root CA (self-signed, in OS/browser trust store)
  └── Intermediate CA (signed by Root CA)
        └── Server Certificate (signed by Intermediate CA)
```

For TLS, the server must present: **Server Cert + Intermediate Cert(s)**  
The Root CA is already trusted by clients.

## Step 1: Assemble the Certificate Chain

```bash
# You should have these files from your CA:

# - server.crt   (your Portainer server certificate)
# - intermediate.crt  (intermediate CA certificate)
# - root.crt    (root CA - usually not included in chain)

# Assemble the full chain (server cert + intermediate)
cat server.crt intermediate.crt > fullchain.pem

# For multi-level chains
cat server.crt intermediate2.crt intermediate1.crt > fullchain.pem
# Note: Order matters - server cert first, then intermediates from leaf to root
```

## Step 2: Verify the Chain is Correct

```bash
# Verify the chain is valid
openssl verify -CAfile root.crt -untrusted intermediate.crt server.crt
# Expected: server.crt: OK

# Check the chain order
openssl crl2pkcs7 -nocrl -certfile fullchain.pem \
  | openssl pkcs7 -print_certs -noout

# Verify certificate relationships
# Subject of each cert should match Issuer of the next
openssl x509 -in server.crt -noout -subject -issuer
openssl x509 -in intermediate.crt -noout -subject -issuer
openssl x509 -in root.crt -noout -subject -issuer
```

## Step 3: Configure Portainer with the Full Chain

```bash
# Copy fullchain to Portainer data volume
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/certs \
  alpine \
  sh -c "mkdir -p /data/certs && \
    cp /certs/fullchain.pem /data/certs/cert.pem && \
    cp /certs/server.key /data/certs/key.pem"

# Deploy Portainer with the chain certificate
docker stop portainer && docker rm portainer

docker run -d \
  -p 9443:9443 \
  -p 8000:8000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --ssl \
  --sslcert /data/certs/cert.pem \
  --sslkey /data/certs/key.pem \
  --http-disabled
```

## Step 4: Verify Chain in Browser and CLI

```bash
# Check the chain Portainer is presenting
echo | openssl s_client -connect portainer.example.com:9443 \
  -showcerts 2>/dev/null | grep -E "subject|issuer"

# Should show all certs in the chain:
# subject=CN=portainer.example.com
# issuer=CN=Intermediate CA, O=MyOrg
# subject=CN=Intermediate CA, O=MyOrg
# issuer=CN=Root CA, O=MyOrg

# Verify no chain errors
echo | openssl s_client -connect portainer.example.com:9443 2>/dev/null \
  | grep -E "Verify return|Certificate chain"
```

## Common Chain Issues and Fixes

### "unable to verify the first certificate" Error

```bash
# This means intermediate cert is missing from the chain
# Fix: Ensure fullchain.pem includes intermediate cert(s)
cat server.crt intermediate.crt > fullchain.pem
```

### Certificate Order Wrong

```bash
# Wrong order causes chain verification failure
# Check order:
openssl crl2pkcs7 -nocrl -certfile fullchain.pem | openssl pkcs7 -print_certs -text -noout | grep "Subject:"

# Correct order: server cert first, then intermediates
```

### Self-Signed Certificate Appearing in Chain

```bash
# This means the chain includes the root CA
# Root CA should NOT be in the chain file - clients already have it
# Remove the last cert from your chain file if it's the root CA
openssl x509 -in root.crt -noout -issuer
# If Issuer == Subject, it's self-signed (root CA)
```

## Using PKCS#12 (PFX) Bundle

```bash
# If your CA provides a .pfx file, extract the components
openssl pkcs12 -in portainer.pfx -nokeys -clcerts -out server.crt
openssl pkcs12 -in portainer.pfx -nokeys -cacerts -out chain.crt
openssl pkcs12 -in portainer.pfx -nocerts -nodes -out server.key

# Assemble chain
cat server.crt chain.crt > fullchain.pem
```

## Conclusion

Proper certificate chain configuration is essential for enterprise PKI environments. The key principle is: the certificate file (`cert.pem`) provided to Portainer must contain the server certificate followed by all intermediate certificates. The root CA is intentionally omitted because clients already trust it. Verify your chain with `openssl s_client` before deploying to production to avoid unexpected trust errors.
