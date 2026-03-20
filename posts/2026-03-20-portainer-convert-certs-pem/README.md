# How to Convert Certificates to PEM Format for Portainer - Certs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, SSL, TLS, PEM, Certificates, OpenSSL, Conversion

Description: A guide to converting SSL/TLS certificates from various formats (DER, PFX/PKCS#12, PKCS#7, Java Keystore) to PEM format for use with Portainer.

## Overview

Portainer requires SSL certificates in PEM format (Base64-encoded DER with `-----BEGIN CERTIFICATE-----` headers). Certificates from enterprise CAs, Windows environments, or Java applications often come in other formats - DER, PKCS#12 (PFX), PKCS#7, or JKS. This guide covers converting all common certificate formats to PEM for use with Portainer.

## Prerequisites

- OpenSSL installed
- The certificate files to convert
- Java `keytool` (for JKS conversion)

## Understanding Certificate Formats

| Format | Extension | Encoding | Common Source |
|---|---|---|---|
| PEM | .pem, .crt, .cer | Base64 | Linux/OpenSSL |
| DER | .der, .cer | Binary | Windows, Java |
| PKCS#12 | .pfx, .p12 | Binary | Windows, IIS |
| PKCS#7 | .p7b, .p7c | Base64 or Binary | Windows CA |
| JKS | .jks, .keystore | Binary | Java |

## How to Identify Your Certificate Format

```bash
# Check if file is PEM (shows text headers)

head -1 certificate.crt
# PEM: -----BEGIN CERTIFICATE-----
# DER/Binary: shows non-text characters

# Check with file command
file certificate.crt
# PEM: ASCII text
# DER: data (binary)
# PKCS#12: data (has magic bytes)

# Try reading with openssl
openssl x509 -in certificate.crt -text -noout 2>/dev/null && echo "PEM format"
openssl x509 -in certificate.crt -inform DER -text -noout 2>/dev/null && echo "DER format"
```

## Convert DER to PEM

```bash
# DER certificate to PEM
openssl x509 -inform DER -in certificate.der -out certificate.pem

# DER private key to PEM
openssl rsa -inform DER -in private.der -out private.pem

# Verify conversion
openssl x509 -in certificate.pem -text -noout | head -5
```

## Convert PKCS#12 (PFX) to PEM

Windows IIS and Azure commonly use PFX format:

```bash
# Extract everything from PFX
openssl pkcs12 -in certificate.pfx -out all-certs.pem -nodes

# Extract only the server certificate (no CA certs, no key)
openssl pkcs12 -in certificate.pfx -nokeys -clcerts -out server-cert.pem

# Extract only CA certificates (intermediate chain)
openssl pkcs12 -in certificate.pfx -nokeys -cacerts -out chain.pem

# Extract only the private key (no certificates)
openssl pkcs12 -in certificate.pfx -nocerts -nodes -out private.key

# Assemble for Portainer
cat server-cert.pem chain.pem > fullchain.pem
```

```bash
# If PFX has a password
openssl pkcs12 -in certificate.pfx -nokeys -clcerts \
  -out server-cert.pem \
  -passin pass:YourPFXPassword
```

## Convert PKCS#7 to PEM

Windows CA often exports in PKCS#7 format:

```bash
# P7B to PEM (certificate chain)
openssl pkcs7 -print_certs -in certificate.p7b -out certificate.pem

# If the P7B is in binary DER format
openssl pkcs7 -inform DER -print_certs -in certificate.p7b -out certificate.pem

# Verify
openssl x509 -in certificate.pem -text -noout | grep -E "Subject:|Issuer:"
```

## Convert Java Keystore (JKS) to PEM

```bash
# Step 1: Convert JKS to PKCS#12
keytool -importkeystore \
  -srckeystore keystore.jks \
  -destkeystore keystore.p12 \
  -deststoretype PKCS12 \
  -srcalias portainer \
  -deststorepass changeit \
  -srcstorepass changeit

# Step 2: Convert PKCS#12 to PEM
openssl pkcs12 -in keystore.p12 \
  -nokeys -clcerts \
  -out server-cert.pem \
  -passin pass:changeit

openssl pkcs12 -in keystore.p12 \
  -nocerts -nodes \
  -out private.key \
  -passin pass:changeit
```

## Verify PEM Files for Portainer

```bash
# Verify certificate is valid PEM
openssl x509 -in server-cert.pem -noout -text | grep -E "Subject:|Valid"

# Verify private key matches certificate
CERT_MODULUS=$(openssl x509 -in server-cert.pem -noout -modulus | md5sum)
KEY_MODULUS=$(openssl rsa -in private.key -noout -modulus | md5sum)

if [ "${CERT_MODULUS}" = "${KEY_MODULUS}" ]; then
  echo "Certificate and key match"
else
  echo "ERROR: Certificate and key do not match!"
fi

# Verify the chain
openssl verify -CAfile chain.pem server-cert.pem
```

## Deploy to Portainer

```bash
# Copy converted PEM files to Portainer
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/certs \
  alpine \
  sh -c "mkdir -p /data/certs && \
    cp /certs/fullchain.pem /data/certs/cert.pem && \
    cp /certs/private.key /data/certs/key.pem && \
    chmod 644 /data/certs/cert.pem && \
    chmod 600 /data/certs/key.pem"

# Restart Portainer
docker restart portainer

# Verify
echo | openssl s_client -connect localhost:9443 2>/dev/null \
  | openssl x509 -noout -subject -dates
```

## Quick Reference

```bash
# DER → PEM cert:    openssl x509 -inform DER -in cert.der -out cert.pem
# DER → PEM key:     openssl rsa -inform DER -in key.der -out key.pem
# PFX → PEM cert:    openssl pkcs12 -in cert.pfx -nokeys -clcerts -out cert.pem
# PFX → PEM key:     openssl pkcs12 -in cert.pfx -nocerts -nodes -out key.pem
# PFX → chain:       openssl pkcs12 -in cert.pfx -nokeys -cacerts -out chain.pem
# P7B → PEM:         openssl pkcs7 -print_certs -in cert.p7b -out cert.pem
# JKS → P12 → PEM:   keytool then openssl (two-step process)
```

## Conclusion

Converting certificates to PEM format is a common prerequisite for deploying Portainer with enterprise PKI certificates. OpenSSL handles most conversions directly, while JKS files require a two-step conversion via PKCS#12. Always verify that the converted certificate and private key match before deploying to Portainer, and ensure the certificate chain is complete to avoid browser trust errors.
