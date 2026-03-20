# How to Set Up mTLS (Mutual TLS) in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, mTLS, Security, Business Edition, TLS

Description: Configure mutual TLS (mTLS) in Portainer Business Edition to require client certificate authentication for the highest level of API and UI security.

---

Mutual TLS (mTLS) requires both the server and the client to present valid certificates, ensuring that only authenticated clients can connect to Portainer. This is the highest level of transport security available for container management APIs.

## Understanding mTLS

In standard TLS, only the server presents a certificate. In mTLS:
- Server presents its certificate to the client
- Client also presents its certificate to the server
- Both must be signed by a trusted CA

## Prerequisites

- A Certificate Authority (CA) certificate
- Server certificate signed by the CA
- Client certificates signed by the same CA

## Step 1: Generate a CA and Certificates

```bash
# Create a directory for CA management

mkdir -p /opt/portainer-ca/{ca,server,client}
cd /opt/portainer-ca

# Generate CA private key and self-signed certificate
openssl genrsa -out ca/ca.key 4096
openssl req -new -x509 -days 3650 -key ca/ca.key \
  -out ca/ca.crt \
  -subj "/C=US/O=MyOrg/CN=Portainer-CA"

# Generate server certificate
openssl genrsa -out server/portainer.key 2048
openssl req -new -key server/portainer.key \
  -out server/portainer.csr \
  -subj "/C=US/O=MyOrg/CN=portainer.example.com"
openssl x509 -req -days 365 \
  -in server/portainer.csr \
  -CA ca/ca.crt \
  -CAkey ca/ca.key \
  -CAcreateserial \
  -out server/portainer.crt

# Generate a client certificate
openssl genrsa -out client/client.key 2048
openssl req -new -key client/client.key \
  -out client/client.csr \
  -subj "/C=US/O=MyOrg/CN=portainer-client"
openssl x509 -req -days 365 \
  -in client/client.csr \
  -CA ca/ca.crt \
  -CAkey ca/ca.key \
  -CAcreateserial \
  -out client/client.crt
```

## Step 2: Configure Portainer with mTLS

```bash
# Start Portainer with mTLS configuration
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /opt/portainer-ca:/certs:ro \
  portainer/portainer-ee:latest \
  --ssl \
  --sslcert /certs/server/portainer.crt \
  --sslkey /certs/server/portainer.key \
  --sslcacert /certs/ca/ca.crt
```

## Step 3: Configure the Portainer Agent with mTLS

When connecting agents, provide the CA cert for agent-server mutual authentication:

```bash
# Portainer Agent with TLS configuration
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /opt/portainer-ca:/certs:ro \
  portainer/agent:latest \
  --sslcert /certs/server/portainer.crt \
  --sslkey /certs/server/portainer.key \
  --sslcacert /certs/ca/ca.crt
```

## Step 4: Use Client Certificate for API Access

```bash
# API call with client certificate authentication
curl -X GET \
  https://localhost:9443/api/status \
  --cert /opt/portainer-ca/client/client.crt \
  --key /opt/portainer-ca/client/client.key \
  --cacert /opt/portainer-ca/ca/ca.crt

echo "mTLS connection successful"
```

## Verify mTLS is Working

```bash
# Try connecting without a client certificate (should fail)
curl -k https://localhost:9443/api/status
# Expected: SSL error - certificate required

# Connect with client certificate (should succeed)
curl https://localhost:9443/api/status \
  --cert /opt/portainer-ca/client/client.crt \
  --key /opt/portainer-ca/client/client.key \
  --cacert /opt/portainer-ca/ca/ca.crt
```

---

*Combine mTLS with [OneUptime](https://oneuptime.com) monitoring for a comprehensive security and observability posture.*
