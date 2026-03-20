# How to Set Up Swarm Inter-Node Encryption in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Encryption, Security, mTLS

Description: Enable and configure Docker Swarm's built-in inter-node encryption for overlay networks managed by Portainer.

## Introduction

Docker Swarm provides built-in encryption for inter-node communication and overlay networks. Enabling encryption ensures that traffic between Swarm nodes and between containers on overlay networks is protected, even in environments without network-level encryption.

## Swarm-Level Encryption

Docker Swarm encrypts two layers automatically when initialized with `--autolock=true`:
1. **Raft logs**: Manager-to-manager communication
2. **Overlay network data plane**: Container-to-container traffic

## Initializing Swarm with Encryption

```bash
# Initialize swarm with autolock (encrypts Raft logs)

docker swarm init \
  --advertise-addr 192.168.1.10 \
  --autolock=true

# Output includes an unlock key:
# Swarm initialized: current node (xxx) is now a manager.
# 
# Manager Unlock Key: SWMKEY-1-xxxxxxxxxxxxxxxx
#
# IMPORTANT: Store the unlock key in a secure location!

# Save the unlock key securely
UNLOCK_KEY="SWMKEY-1-xxxxxxxxxxxxxxxx"
echo $UNLOCK_KEY | vault write secret/swarm-unlock-key value=-
```

## Unlocking Swarm After Restart

When autolock is enabled, managers must be unlocked after Docker restarts:

```bash
# When Docker restarts, the Swarm manager is locked
# Unlock it with the stored key
docker swarm unlock
# Enter the unlock key when prompted

# Or provide it non-interactively
echo $UNLOCK_KEY | docker swarm unlock

# Verify the manager is unlocked
docker node ls
```

## Rotating the Unlock Key

```bash
# Rotate the unlock key periodically
docker swarm unlock-key --rotate

# Display the current unlock key
docker swarm unlock-key
```

## Overlay Network Encryption

Enable encryption for overlay networks to protect container-to-container traffic:

```yaml
# encrypted-network-stack.yml
version: '3.8'

services:
  frontend:
    image: nginx:latest
    deploy:
      replicas: 2
    networks:
      - frontend-net

  api:
    image: myapp:latest
    deploy:
      replicas: 3
    networks:
      - frontend-net
      - backend-net

  db:
    image: postgres:15
    deploy:
      replicas: 1
    networks:
      - backend-net
    environment:
      POSTGRES_PASSWORD: secret

networks:
  frontend-net:
    driver: overlay
    driver_opts:
      encrypted: "true"  # Enable IPSEC encryption
  
  backend-net:
    driver: overlay
    driver_opts:
      encrypted: "true"   # Encrypted backend traffic
```

## Verifying Network Encryption

```bash
# Create a test encrypted network
docker network create \
  --driver overlay \
  --opt encrypted \
  --attachable \
  secure-test-net

# Inspect the network to verify encryption
docker network inspect secure-test-net \
  --format '{{.Options}}'
# Should show: map[encrypted:true]

# Test connectivity (encrypted traffic)
docker run --rm \
  --network secure-test-net \
  alpine ping -c 3 other-container
```

## Securing the Docker Daemon API

```bash
# Generate CA, server, and client certificates for Docker daemon TLS
mkdir -p ~/docker-tls && cd ~/docker-tls

# CA
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -out ca.pem -subj "/CN=docker-ca"

# Server
openssl genrsa -out server-key.pem 4096
openssl req -new -key server-key.pem -out server.csr -subj "/CN=swarm-manager"
openssl x509 -req -days 365 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem \
  -extfile <(echo "subjectAltName=IP:192.168.1.10,IP:127.0.0.1")

# Configure Docker daemon with TLS
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "tls": true,
  "tlsverify": true,
  "tlscacert": "/etc/docker/tls/ca.pem",
  "tlscert": "/etc/docker/tls/server-cert.pem",
  "tlskey": "/etc/docker/tls/server-key.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
}
EOF

sudo systemctl restart docker
```

## Portainer with Encrypted Swarm

Portainer works transparently with encrypted Swarm clusters:

```bash
# Connect Portainer to TLS-secured Swarm
docker run -d \
  --name portainer \
  -p 9443:9443 \
  -v portainer_data:/data \
  -v /etc/docker/tls:/certs:ro \
  portainer/portainer-ce:latest \
  --ssl \
  --sslcert /certs/server-cert.pem \
  --sslkey /certs/server-key.pem
```

## Conclusion

Docker Swarm's built-in encryption capabilities protect both management plane traffic (Raft logs with autolock) and data plane traffic (encrypted overlay networks). Portainer manages encrypted Swarm clusters transparently, displaying services and containers without requiring knowledge of the underlying encryption. Enabling encryption at the network layer provides defense-in-depth for containerized workloads in shared or untrusted network environments.
