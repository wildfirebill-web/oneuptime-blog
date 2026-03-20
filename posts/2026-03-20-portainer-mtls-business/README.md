# How to Configure mTLS for Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, mtls, mutual-tls, security, business-edition, certificates

Description: A guide to configuring mutual TLS (mTLS) authentication for Portainer Business Edition to enforce client certificate authentication.

## Overview

Mutual TLS (mTLS) requires both the server and the client to present certificates, providing stronger authentication than username/password alone. Portainer Business Edition supports mTLS for securing the communication between Portainer and connected agents, as well as for Docker endpoints. This guide covers configuring mTLS for agent communication and Docker TLS.

## Prerequisites

- Portainer Business Edition
- A Certificate Authority (CA) for issuing client and server certificates
- OpenSSL for certificate generation

## Understanding mTLS in Portainer Context

Portainer uses TLS in two contexts:
1. **Web UI/API TLS**: Portainer server certificate for browser/API connections
2. **Docker Endpoint TLS**: Portainer connects to Docker daemons using TLS client certs

## Step 1: Create a Certificate Authority

```bash
mkdir -p /opt/mtls-ca/{certs,keys,requests}

# Generate CA key
openssl genrsa -out /opt/mtls-ca/keys/ca.key 4096

# Generate CA certificate
openssl req -new -x509 -days 3650 \
  -key /opt/mtls-ca/keys/ca.key \
  -out /opt/mtls-ca/certs/ca.crt \
  -subj "/C=US/ST=CA/O=MyOrg/CN=PortainerCA"
```

## Step 2: Generate Server Certificate (Portainer)

```bash
# Server key
openssl genrsa -out /opt/mtls-ca/keys/portainer-server.key 2048

# Server CSR
openssl req -new \
  -key /opt/mtls-ca/keys/portainer-server.key \
  -out /opt/mtls-ca/requests/portainer-server.csr \
  -subj "/CN=portainer.example.com" \
  -addext "subjectAltName=DNS:portainer.example.com"

# Sign server cert with CA
openssl x509 -req -days 365 \
  -in /opt/mtls-ca/requests/portainer-server.csr \
  -CA /opt/mtls-ca/certs/ca.crt \
  -CAkey /opt/mtls-ca/keys/ca.key \
  -CAcreateserial \
  -out /opt/mtls-ca/certs/portainer-server.crt \
  -extfile <(echo "subjectAltName=DNS:portainer.example.com")
```

## Step 3: Generate Client Certificate

```bash
# Client key
openssl genrsa -out /opt/mtls-ca/keys/portainer-client.key 2048

# Client CSR
openssl req -new \
  -key /opt/mtls-ca/keys/portainer-client.key \
  -out /opt/mtls-ca/requests/portainer-client.csr \
  -subj "/CN=portainer-client/O=MyOrg"

# Sign client cert
openssl x509 -req -days 365 \
  -in /opt/mtls-ca/requests/portainer-client.csr \
  -CA /opt/mtls-ca/certs/ca.crt \
  -CAkey /opt/mtls-ca/keys/ca.key \
  -CAcreateserial \
  -out /opt/mtls-ca/certs/portainer-client.crt
```

## Step 4: Configure Docker Daemon for TLS

On each Docker host that Portainer will connect to:

```bash
# Configure Docker daemon to require TLS client certs
sudo mkdir -p /etc/docker/tls

# Copy CA cert and server certs
sudo cp /opt/mtls-ca/certs/ca.crt /etc/docker/tls/
sudo cp /opt/mtls-ca/certs/docker-server.crt /etc/docker/tls/
sudo cp /opt/mtls-ca/keys/docker-server.key /etc/docker/tls/

# Configure Docker daemon
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "tls": true,
  "tlscacert": "/etc/docker/tls/ca.crt",
  "tlscert": "/etc/docker/tls/docker-server.crt",
  "tlskey": "/etc/docker/tls/docker-server.key",
  "tlsverify": true,
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
}
EOF

sudo systemctl restart docker
```

## Step 5: Add TLS Docker Endpoint to Portainer

Via Portainer UI:
1. Navigate to **Environments** → **Add environment**
2. Select **Docker Standalone**
3. Enter the Docker host URL: `tcp://docker-host:2376`
4. Enable **TLS**
5. Upload:
   - **TLS CA certificate**: `ca.crt`
   - **TLS certificate**: `portainer-client.crt`
   - **TLS key**: `portainer-client.key`
6. Click **Connect**

Via API:
```bash
curl -X POST \
  "https://portainer.example.com:9443/api/endpoints" \
  -H "Authorization: Bearer ${TOKEN}" \
  -F "Name=tls-docker-host" \
  -F "EndpointCreationType=1" \
  -F "URL=tcp://docker-host:2376" \
  -F "TLS=true" \
  -F "TLSSkipVerify=false" \
  -F "TLSCACert=@/opt/mtls-ca/certs/ca.crt" \
  -F "TLSCert=@/opt/mtls-ca/certs/portainer-client.crt" \
  -F "TLSKey=@/opt/mtls-ca/keys/portainer-client.key"
```

## Verify mTLS Connection

```bash
# Test Docker TLS connection manually
docker --tlsverify \
  --tlscacert=/opt/mtls-ca/certs/ca.crt \
  --tlscert=/opt/mtls-ca/certs/portainer-client.crt \
  --tlskey=/opt/mtls-ca/keys/portainer-client.key \
  -H tcp://docker-host:2376 \
  version

# Check that connection without client cert fails
docker --tls \
  --tlscacert=/opt/mtls-ca/certs/ca.crt \
  -H tcp://docker-host:2376 \
  version
# Expected: Error response: tls: certificate required
```

## Conclusion

mTLS between Portainer and Docker endpoints provides strong authentication — only Portainer with the correct client certificate can manage the Docker daemon. This prevents unauthorized access even if an attacker has network access to port 2376. Always keep CA keys secure, rotate certificates before expiry, and revoke compromised certificates by reissuing and distributing updated CRLs or OCSP.
