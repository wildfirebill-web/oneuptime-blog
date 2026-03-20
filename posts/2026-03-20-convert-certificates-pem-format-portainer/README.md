# How to Convert Certificates to PEM Format for Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, SSL, PEM, Certificate Conversion, OpenSSL

Description: Learn how to convert SSL certificates from various formats (DER, PFX, P12, PKCS7) to PEM format required by Portainer.

---

Portainer requires certificates and keys in PEM format (base64-encoded with `-----BEGIN CERTIFICATE-----` headers). Certificates from Windows environments, Java keystores, or commercial CAs often come in different formats that need conversion.

## Common Certificate Formats

| Format | Extension | Description |
|--------|-----------|-------------|
| PEM | .pem, .crt, .cer, .key | Base64-encoded, Portainer-compatible |
| DER | .der, .cer | Binary-encoded, needs conversion |
| PKCS#12 | .pfx, .p12 | Bundle with cert and key |
| PKCS#7 | .p7b, .p7c | Chain bundle, no private key |

## Convert DER to PEM

```bash
# Convert DER-encoded certificate to PEM
openssl x509 \
  -inform der \
  -in certificate.der \
  -out certificate.pem

# Convert DER-encoded private key to PEM
openssl rsa \
  -inform der \
  -in private.der \
  -out private.pem

# Verify the converted certificate
openssl x509 -in certificate.pem -noout -text | head -20
```

## Convert PKCS#12 (PFX/P12) to PEM

Windows environments commonly export certificates as PFX files:

```bash
# Extract certificate from PFX (will prompt for PFX password)
openssl pkcs12 \
  -in certificate.pfx \
  -nokeys \
  -out certificate.pem

# Extract private key from PFX (no passphrase on output)
openssl pkcs12 \
  -in certificate.pfx \
  -nocerts \
  -nodes \
  -out private.key

# Extract both in one command (creates combined PEM)
openssl pkcs12 \
  -in certificate.pfx \
  -nodes \
  -out combined.pem

# Then split combined.pem:
# Extract just the certificate
openssl x509 -in combined.pem -out certificate.pem

# Extract just the key
openssl pkey -in combined.pem -out private.key
```

## Convert PKCS#7 (P7B) to PEM

```bash
# Convert PKCS#7 bundle to PEM format
openssl pkcs7 \
  -print_certs \
  -in certificate.p7b \
  -out certificate.pem

# Verify the conversion
openssl x509 -in certificate.pem -noout -subject
```

## Verify PEM Format

Before using with Portainer, verify your PEM files are correctly formatted:

```bash
# Check certificate PEM format
openssl x509 -in certificate.pem -noout -text | grep "Subject:\|Issuer:\|Not After"

# Check private key PEM format
openssl rsa -in private.key -check -noout
# Expected: RSA key ok

# Verify the key matches the certificate
CERT_MOD=$(openssl x509 -noout -modulus -in certificate.pem | openssl md5)
KEY_MOD=$(openssl rsa -noout -modulus -in private.key | openssl md5)
[ "$CERT_MOD" = "$KEY_MOD" ] && echo "Key matches certificate" || echo "ERROR: Key does not match"
```

## Deploy to Portainer

After converting to PEM format:

```bash
# Copy converted files to Portainer cert directory
cp certificate.pem /opt/portainer/certs/portainer.crt
cp private.key /opt/portainer/certs/portainer.key
chmod 600 /opt/portainer/certs/portainer.key

# Restart Portainer with the converted certificates
docker restart portainer
```

## Create a PEM with Full Chain

If your PEM file contains only the server certificate, add intermediates:

```bash
# Concatenate server cert + intermediates into a fullchain PEM
cat certificate.pem intermediate.pem > fullchain.pem

# Use fullchain.pem as the --sslcert argument
```

---

*Monitor certificate expiry dates across all your services with [OneUptime](https://oneuptime.com) SSL monitoring.*
