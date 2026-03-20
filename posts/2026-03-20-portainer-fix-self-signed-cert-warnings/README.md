# How to Fix Self-Signed Certificate Warnings in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, SSL, TLS, Self-Signed, Certificates, Browser-warning

Description: A guide to eliminating browser self-signed certificate warnings when accessing Portainer, with options ranging from proper CA-signed certs to trusting an internal CA.

## Overview

Portainer ships with a self-signed certificate that causes "Your connection is not private" warnings in browsers. While you can click through these warnings, they're unprofessional and can confuse users. This guide covers all approaches to fix these warnings: using Let's Encrypt, creating an internal CA, and distributing that CA to client machines.

## Prerequisites

- Portainer running with default or custom SSL
- Administrative access to client machines (for CA distribution)

## Option 1: Use Let's Encrypt (Best for Public-Facing Portainer)

The simplest fix for internet-accessible Portainer:

```bash
# Obtain Let's Encrypt certificate

sudo certbot certonly --standalone \
  -d portainer.example.com \
  --agree-tos -m admin@example.com --non-interactive

# Deploy to Portainer
docker run --rm \
  -v portainer_data:/data \
  -v /etc/letsencrypt/live/portainer.example.com:/certs:ro \
  alpine \
  sh -c "mkdir -p /data/certs && cp /certs/fullchain.pem /data/certs/cert.pem && cp /certs/privkey.pem /data/certs/key.pem"

docker restart portainer
```

## Option 2: Internal CA + Trust Distribution (Best for Private Networks)

### Step 1: Create Internal CA

```bash
# Generate CA key and certificate
openssl genrsa -out /opt/internal-ca/ca.key 4096
openssl req -new -x509 -days 3650 \
  -key /opt/internal-ca/ca.key \
  -out /opt/internal-ca/ca.crt \
  -subj "/C=US/O=MyOrg/CN=MyOrg Internal CA"
```

### Step 2: Generate Server Certificate Signed by Internal CA

```bash
openssl req -newkey rsa:2048 -nodes \
  -keyout /opt/internal-ca/portainer.key \
  -out /opt/internal-ca/portainer.csr \
  -subj "/CN=portainer.internal.example.com"

# Sign with SANs
openssl x509 -req -days 825 \
  -in /opt/internal-ca/portainer.csr \
  -CA /opt/internal-ca/ca.crt \
  -CAkey /opt/internal-ca/ca.key \
  -CAcreateserial \
  -out /opt/internal-ca/portainer.crt \
  -extfile <(printf "subjectAltName=DNS:portainer.internal.example.com,IP:$(hostname -I | awk '{print $1}')")

# Deploy to Portainer
docker run --rm \
  -v portainer_data:/data \
  -v /opt/internal-ca:/certs \
  alpine \
  sh -c "mkdir -p /data/certs && cp /certs/portainer.crt /data/certs/cert.pem && cp /certs/portainer.key /data/certs/key.pem"

docker restart portainer
```

### Step 3: Distribute CA Certificate to Client Machines

#### Ubuntu/Debian

```bash
sudo cp /opt/internal-ca/ca.crt /usr/local/share/ca-certificates/myorg-ca.crt
sudo update-ca-certificates
```

#### RHEL/Rocky/Oracle/CentOS

```bash
sudo cp /opt/internal-ca/ca.crt /etc/pki/ca-trust/source/anchors/myorg-ca.crt
sudo update-ca-trust
```

#### macOS

```bash
# Add to System keychain (requires admin)
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain /opt/internal-ca/ca.crt
```

#### Windows (Group Policy or Manual)

```powershell
# Import CA cert to Trusted Root Certification Authorities
Import-Certificate -FilePath "C:\myorg-ca.crt" \
  -CertStoreLocation Cert:\LocalMachine\Root
```

## Option 3: Use Nginx with Trusted Certificate

Configure Nginx as a reverse proxy with a trusted cert while Portainer uses its default self-signed cert internally:

```bash
# Nginx handles trusted HTTPS externally
# Portainer remains on self-signed internally
# Nginx connects to Portainer with --proxy-ssl-verify off
```

## Verifying the Fix

```bash
# After applying certificate changes:

# Test from command line (no -k needed)
curl https://portainer.internal.example.com:9443/api/status

# Check certificate is trusted
openssl verify -CAfile /opt/internal-ca/ca.crt /opt/internal-ca/portainer.crt
# Expected: portainer.crt: OK

# Check what browsers see
echo | openssl s_client -connect portainer.internal.example.com:9443 2>/dev/null \
  | grep -E "Verify return code"
# Expected: Verify return code: 0 (ok)
```

## Chrome/Firefox-Specific Fixes

For browsers that maintain their own cert store (separate from OS):

```bash
# Chrome: Import CA via Settings → Privacy & Security → Manage Certificates
# Then import your CA cert as a trusted Authority

# Firefox: Settings → Privacy & Security → Certificates → View Certificates
# Authorities tab → Import your CA cert
```

## Conclusion

Self-signed certificate warnings can be fixed permanently by using Let's Encrypt (for public Portainer) or deploying an internal CA and distributing it to all client machines (for private Portainer). The internal CA approach is the enterprise standard for private services. Once the CA is trusted by client machines, users will never see warnings again, and scripts using `curl` will work without `--insecure` flags.
