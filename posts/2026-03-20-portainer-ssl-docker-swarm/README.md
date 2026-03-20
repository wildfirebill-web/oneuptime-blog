# How to Configure SSL/TLS for Portainer on Docker Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, ssl, tls, docker-swarm, security, certificates

Description: A guide to configuring SSL/TLS certificates for Portainer deployed on Docker Swarm using Docker Secrets for certificate management.

## Overview

When Portainer is deployed on Docker Swarm, certificate management takes advantage of Docker Secrets — an encrypted store for sensitive data. This approach is more secure than bind-mounting certificate files and integrates with Swarm's native secret management. This guide covers deploying Portainer on Swarm with proper SSL/TLS certificates.

## Prerequisites

- Docker Swarm initialized (`docker swarm init`)
- SSL certificates (self-signed or CA-signed)
- Admin access to the Swarm manager

## Step 1: Generate SSL Certificates

```bash
# Generate certificates (same as standalone approach)
mkdir -p /opt/portainer/certs

openssl req -newkey rsa:2048 -nodes \
  -keyout /opt/portainer/certs/portainer.key \
  -x509 -days 365 \
  -out /opt/portainer/certs/portainer.crt \
  -subj "/CN=portainer.example.com" \
  -addext "subjectAltName=DNS:portainer.example.com,IP:$(hostname -I | awk '{print $1}')"
```

## Step 2: Create Docker Secrets for Certificates

```bash
# Create secrets from certificate files
docker secret create portainer-ssl-cert /opt/portainer/certs/portainer.crt
docker secret create portainer-ssl-key /opt/portainer/certs/portainer.key

# Verify secrets
docker secret ls
```

## Step 3: Deploy Portainer Stack with SSL

```yaml
# portainer-ssl-stack.yml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    command:
      - --ssl
      - --sslcert=/run/secrets/portainer-ssl-cert
      - --sslkey=/run/secrets/portainer-ssl-key
      - --http-disabled
    secrets:
      - portainer-ssl-cert
      - portainer-ssl-key
    ports:
      - target: 9443
        published: 9443
        protocol: tcp
        mode: host
      - target: 8000
        published: 8000
        protocol: tcp
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints:
          - node.platform.os == linux

secrets:
  portainer-ssl-cert:
    external: true
  portainer-ssl-key:
    external: true

volumes:
  portainer_data:

networks:
  agent_network:
    driver: overlay
    attachable: true
```

```bash
docker stack deploy -c portainer-ssl-stack.yml portainer
```

## Step 4: Verify Deployment

```bash
# Check stack status
docker stack ps portainer

# Verify certificate in use
echo | openssl s_client -connect localhost:9443 2>/dev/null \
  | openssl x509 -noout -dates -subject
```

## Rotating Certificates on Swarm

```bash
# Create new secret version
docker secret create portainer-ssl-cert-v2 /opt/portainer/certs/new-portainer.crt
docker secret create portainer-ssl-key-v2 /opt/portainer/certs/new-portainer.key

# Update service to use new secrets
docker service update \
  --secret-rm portainer-ssl-cert \
  --secret-rm portainer-ssl-key \
  --secret-add source=portainer-ssl-cert-v2,target=portainer-ssl-cert \
  --secret-add source=portainer-ssl-key-v2,target=portainer-ssl-key \
  portainer_portainer

# Verify
docker service ps portainer_portainer
```

## Using Nginx as SSL Terminator for Swarm

```yaml
# Add Nginx as SSL terminator in the stack
services:
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    configs:
      - source: nginx-config
        target: /etc/nginx/conf.d/default.conf
    secrets:
      - portainer-ssl-cert
      - portainer-ssl-key
    networks:
      - agent_network
    deploy:
      placement:
        constraints:
          - node.role == manager

configs:
  nginx-config:
    external: true
```

## Conclusion

Using Docker Secrets for certificate management in Swarm provides encrypted storage and seamless rotation without container restarts. The `--secret-add` / `--secret-rm` workflow allows zero-downtime certificate rotation. For production Swarm deployments, consider automating certificate renewal with cert-manager or Certbot and scripting the secret rotation process.
