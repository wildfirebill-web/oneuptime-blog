# How to Configure CA Files for Helm Repositories in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, Security, TLS

Description: Learn how to configure CA certificate files for private Helm repositories with self-signed TLS certificates in Portainer to enable secure chart fetching.

## Introduction

When hosting a private Helm repository with a self-signed or internal CA-signed TLS certificate, Portainer needs the CA certificate to verify the repository's identity. Without it, Portainer will fail to fetch the chart index with a TLS verification error. This guide covers how to configure CA files for Helm repositories in Portainer.

## Prerequisites

- Portainer CE or BE with a Kubernetes environment
- A private Helm repository with TLS enabled
- The CA certificate (PEM format) that signed the repository's TLS certificate
- Admin access to Portainer

## When Is a CA File Required?

You need a CA file when your Helm repository uses:

- A self-signed TLS certificate
- A certificate signed by an internal/private CA
- A certificate from a CA not included in the default trust store

You do NOT need a CA file for repositories using certificates from public CAs (Let's Encrypt, DigiCert, etc.) since those are trusted by default.

## Step 1: Obtain Your CA Certificate

Get your CA certificate in PEM format:

```bash
# Extract CA cert from your server (if accessible)
openssl s_client -connect helm.internal.company.com:443 -showcerts </dev/null 2>/dev/null | \
  openssl x509 -outform PEM > internal-ca.pem

# Verify the certificate
openssl x509 -in internal-ca.pem -text -noout | grep -E "(Issuer|Subject|Not)"

# Convert DER format to PEM if needed
openssl x509 -inform DER -in ca.der -out ca.pem
```

## Step 2: Configure CA File in Portainer UI

1. Log into Portainer as admin.
2. Select your Kubernetes environment.
3. Click the **gear icon** to open environment settings.
4. Scroll to the **Helm repository** section.
5. Click **Add a Helm repository**.
6. Enter the repository URL (e.g., `https://helm.internal.company.com`).
7. In the **TLS** section, toggle **Use TLS**.
8. Check **Skip TLS verification** if you want to skip verification (NOT recommended for production).
9. For the preferred approach, upload the CA certificate by clicking **CA certificate** and pasting the PEM content.
10. Click **Save**.

## Step 3: Configure CA via the Portainer API

```bash
# Read your CA certificate
CA_CERT=$(cat internal-ca.pem)

# Authenticate
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# Add Helm repo with CA certificate
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/1/kubernetes/helm/repositories" \
  -d "{
    \"url\": \"https://helm.internal.company.com\",
    \"name\": \"internal-repo\",
    \"TLSConfig\": {
      \"TLSCACert\": $(echo "$CA_CERT" | jq -Rs .)
    }
  }"
```

## Step 4: Deploy ChartMuseum with TLS

If you are running ChartMuseum as your private Helm repository, here is a Docker Compose example with TLS:

```yaml
# docker-compose.yml — ChartMuseum with TLS
version: "3.8"
services:
  chartmuseum:
    image: ghcr.io/helm/chartmuseum:v0.16.1
    container_name: chartmuseum
    ports:
      - "443:8443"
    volumes:
      - ./charts:/charts                    # Chart storage directory
      - ./certs/server.crt:/certs/tls.crt  # Server certificate
      - ./certs/server.key:/certs/tls.key  # Server private key
    environment:
      STORAGE: local
      STORAGE_LOCAL_ROOTDIR: /charts
      TLS_CERT: /certs/tls.crt
      TLS_KEY: /certs/tls.key
      DEBUG: "true"
    restart: unless-stopped
```

## Step 5: Generate a Self-Signed CA and Certificate

For testing or internal use, generate your own CA and sign a certificate:

```bash
# Step 1: Generate CA key and certificate
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=Internal Helm CA/O=My Company"

# Step 2: Generate server key and CSR
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
  -subj "/CN=helm.internal.company.com"

# Step 3: Sign the server certificate with your CA
cat > server-ext.cnf << EOF
subjectAltName = DNS:helm.internal.company.com
EOF

openssl x509 -req -days 365 -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -extfile server-ext.cnf

# ca.crt is what you upload to Portainer as the CA file
# server.crt and server.key go on your ChartMuseum server
```

## Step 6: Verify TLS Connectivity

Before configuring Portainer, verify TLS works:

```bash
# Test with curl using your CA cert
curl --cacert internal-ca.pem https://helm.internal.company.com/index.yaml

# Add the Helm repo manually to verify it works
helm repo add internal-repo https://helm.internal.company.com \
  --ca-file internal-ca.pem

helm repo update
helm search repo internal-repo/
```

## Troubleshooting TLS Issues

```bash
# Common error: certificate signed by unknown authority
# Solution: Upload your CA cert to Portainer

# Common error: certificate has expired
openssl x509 -in server.crt -noout -dates

# Check certificate chain
openssl s_client -connect helm.internal.company.com:443 -CAfile ca.pem

# Verify hostname matches CN or SAN
openssl x509 -in server.crt -text -noout | grep -A 3 "Subject Alternative"
```

## Conclusion

Configuring CA files for Helm repositories in Portainer is essential when working with private or self-signed TLS certificates. Upload your CA certificate through the Portainer UI or API to establish trusted connections to internal chart repositories. This approach maintains full TLS verification while supporting internal PKI infrastructure — always prefer proper CA configuration over disabling TLS verification.
