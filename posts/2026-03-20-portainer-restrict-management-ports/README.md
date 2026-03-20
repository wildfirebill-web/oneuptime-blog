# How to Restrict Management Ports in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Networking, Hardening, Firewall

Description: Learn how to restrict access to Portainer's management ports and Docker API ports to prevent unauthorized access to your container management interface.

## Introduction

Portainer and Docker expose several management ports that, if left unrestricted, create significant security risks. This guide covers which ports to protect, how to restrict them using firewall rules, and how to configure Portainer to bind only to specific network interfaces.

## Management Ports Overview

| Port | Service | Risk Level |
|------|---------|------------|
| `9443` | Portainer HTTPS | High - full container management |
| `9000` | Portainer HTTP | Critical - unencrypted management |
| `8000` | Portainer Edge Agent | Medium - for edge environments |
| `2375` | Docker API (no TLS) | Critical - root-equivalent access |
| `2376` | Docker API (TLS) | High - root-equivalent with auth |
| `2377` | Docker Swarm manager | High - Swarm cluster management |

## Step 1: Close Unneeded Ports

First, identify which ports are currently open:

```bash
# Check listening ports

ss -tlnp | grep -E "(9443|9000|8000|2375|2376|2377)"

# Check from external perspective
nmap -p 9000,9443,2375,2376 your-server-ip
```

## Step 2: Firewall Rules with UFW

```bash
# Default: deny all incoming
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH from your IP only
sudo ufw allow from YOUR_ADMIN_IP to any port 22

# Allow Portainer HTTPS ONLY from VPN/office IP range
sudo ufw allow from 10.0.0.0/8 to any port 9443 comment "Portainer HTTPS - internal only"

# NEVER allow these without IP restriction:
# sudo ufw allow 9000   # Plain HTTP - dangerous!
# sudo ufw allow 2375   # Docker API without TLS - extremely dangerous!

# Allow Docker TLS only if needed from specific IPs
sudo ufw allow from 10.0.1.0/24 to any port 2376 comment "Docker TLS - CI/CD subnet only"

# Allow Portainer Edge Agent if using edge environments
sudo ufw allow from 0.0.0.0/0 to any port 8000 comment "Edge Agent tunnel"

# Deny all other management ports explicitly
sudo ufw deny 9000  comment "Block plain HTTP Portainer"
sudo ufw deny 2375  comment "Block Docker API without TLS"

# Enable firewall
sudo ufw enable
sudo ufw reload

# Verify
sudo ufw status verbose
```

## Step 3: iptables Rules (Alternative)

```bash
# Create a new chain for Portainer rules
iptables -N PORTAINER_ACCESS

# Allow from specific IP range
iptables -A PORTAINER_ACCESS -s 10.0.0.0/8 -j ACCEPT     # Internal network
iptables -A PORTAINER_ACCESS -s 172.16.0.0/12 -j ACCEPT   # Private network
iptables -A PORTAINER_ACCESS -j DROP                        # Drop all others

# Apply to Portainer HTTPS port
iptables -A INPUT -p tcp --dport 9443 -j PORTAINER_ACCESS

# Block plain HTTP completely
iptables -A INPUT -p tcp --dport 9000 -j DROP

# Block unauthenticated Docker API
iptables -A INPUT -p tcp --dport 2375 -j DROP

# Save rules
sudo iptables-save > /etc/iptables/rules.v4
```

## Step 4: Bind Portainer to Specific Interface

Prevent Portainer from listening on all interfaces:

```bash
# Bind to internal/VPN IP only
INTERNAL_IP="10.0.0.100"

docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p ${INTERNAL_IP}:9443:9443 \  # Only listen on internal IP
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# This means portainer is NOT accessible via public IP at all
```

For Docker Compose:

```yaml
# docker-compose.yml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    ports:
      - "10.0.0.100:9443:9443"  # Bind to specific IP
      # NOT: - "9443:9443"        This binds to all interfaces
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

## Step 5: Disable Docker TCP Port Completely

If you're not using the Docker TCP API (using socket instead), disable it:

```bash
# Check if Docker is listening on TCP
ss -tlnp | grep docker

# Ensure Docker daemon config does NOT have tcp listener
cat /etc/docker/daemon.json

# If present, remove "tcp://0.0.0.0:2375" from the hosts configuration
sudo nano /etc/docker/daemon.json

# After removing:
# {
#   "log-driver": "json-file",
#   "log-opts": {"max-size": "10m", "max-file": "3"}
#   # NO tcp host
# }

sudo systemctl restart docker
```

## Step 6: Use Docker Socket Proxy

If multiple services need Docker access, use a socket proxy instead of exposing the raw socket:

```yaml
# docker-compose.yml with socket proxy
version: "3.8"
services:
  docker-proxy:
    image: tecnativa/docker-socket-proxy:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      CONTAINERS: 1
      IMAGES: 1
      NETWORKS: 1
      VOLUMES: 1
      INFO: 1
      POST: 0      # Disable write operations for read-only access
    networks:
      - proxy-net
    restart: unless-stopped

  portainer:
    image: portainer/portainer-ce:latest
    environment:
      - DOCKER_HOST=tcp://docker-proxy:2375  # Use the proxy
    depends_on:
      - docker-proxy
    networks:
      - proxy-net
    ports:
      - "10.0.0.100:9443:9443"

networks:
  proxy-net:
    internal: true  # No external access
```

## Step 7: Port Scan Verification

After applying restrictions, verify from an external machine:

```bash
# From outside your network (or use nmap online scanner)
nmap -p 9000,9443,2375,2376,2377 your-public-ip

# Expected results:
# 9000/tcp  closed  portainer-http   # Good: closed
# 9443/tcp  filtered portainer-https # Good: filtered (dropped, not rejected)
# 2375/tcp  closed  docker-api-plain # Good: closed
# 2376/tcp  filtered docker-api-tls  # Good: filtered if not needed

# From inside your VPN/internal network:
nmap -p 9443 10.0.0.100

# Expected:
# 9443/tcp open   https    # Good: accessible from internal network
```

## Conclusion

Restricting Portainer management ports is a critical security measure. Bind Portainer to internal or VPN-only interfaces, use firewall rules to restrict access by source IP, disable the plain HTTP port entirely, and never expose the Docker API TCP port without TLS. Combine port restrictions with VPN-only access for maximum protection - ports that are not reachable from the internet cannot be exploited.
