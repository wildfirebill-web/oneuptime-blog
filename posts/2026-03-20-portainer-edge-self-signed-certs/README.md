# How to Configure Edge Agent with Self-Signed Certificates - Portainer Certs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, TLS, Self-Signed Certificates, Security

Description: Configure the Portainer Edge Agent to trust self-signed certificates when connecting to a Portainer server that uses a self-signed TLS certificate.

## Introduction

When Portainer is deployed with a self-signed TLS certificate, Edge Agents cannot connect because they cannot verify the certificate. This guide covers configuring the Edge Agent to trust your self-signed certificate or skip certificate verification for development environments.

## Option 1: Provide the CA Certificate (Recommended)

Trust the certificate explicitly by providing the CA certificate to the edge agent:

```bash
# Export your self-signed CA certificate

# If using the default Portainer self-signed cert:
openssl s_client -connect portainer.example.com:9443 < /dev/null 2>/dev/null \
  | openssl x509 > portainer-ca.pem

# Verify the certificate
openssl x509 -in portainer-ca.pem -text -noout | grep -E "Subject:|Not After:"
```

Mount the CA certificate and configure the agent:

```bash
docker run -d \
  --name portainer_edge_agent \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /var/run/portainer:/var/run/portainer \
  -v /path/to/portainer-ca.pem:/certs/ca.pem:ro \
  -e EDGE=1 \
  -e EDGE_ID=device-id \
  -e EDGE_KEY=edge-key \
  -e CA_CERT=/certs/ca.pem \
  portainer/agent:latest
```

## Option 2: Skip Certificate Verification (Development Only)

```bash
docker run -d \
  --name portainer_edge_agent \
  -e EDGE=1 \
  -e EDGE_ID=device-id \
  -e EDGE_KEY=edge-key \
  -e EDGE_INSECURE_POLL=1 \    # Skip TLS verification
  portainer/agent:latest
```

**WARNING**: `EDGE_INSECURE_POLL=1` disables TLS certificate validation. Only use in isolated development environments.

## Option 3: Trust System CA Store

If your CA certificate is in the system's trust store:

```bash
# Add CA to system trust store (Ubuntu/Debian)
sudo cp portainer-ca.pem /usr/local/share/ca-certificates/portainer.crt
sudo update-ca-certificates

# The Docker daemon and containers will use the updated trust store
```

## Docker Compose with CA Certificate

```yaml
version: "3.8"

services:
  edge-agent:
    image: portainer/agent:latest
    container_name: portainer_edge_agent
    restart: always
    environment:
      EDGE: "1"
      EDGE_ID: "remote-device"
      EDGE_KEY: "your-edge-key"
      CA_CERT: "/certs/ca.pem"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
      - /var/run/portainer:/var/run/portainer
      - ./portainer-ca.pem:/certs/ca.pem:ro
```

## Verifying Certificate Trust

```bash
# Test that the agent can connect to Portainer with the provided cert
docker logs portainer_edge_agent | grep -E "TLS|cert|connect|error"

# Manual test
curl --cacert portainer-ca.pem https://portainer.example.com/api/system/version
```

## Using Let's Encrypt Instead

The cleanest long-term solution is to use a publicly trusted certificate:

```bash
# Use Traefik with Let's Encrypt
# Edge agents trust Let's Encrypt certs without any extra configuration
# No EDGE_INSECURE_POLL or CA_CERT needed
```

## Conclusion

Self-signed certificates require explicit trust configuration in edge agents. The CA certificate mounting approach maintains security by validating the certificate chain. For production, strongly consider using a properly issued certificate (Let's Encrypt, internal CA, commercial) to eliminate the self-signed certificate complexity across all edge devices.
