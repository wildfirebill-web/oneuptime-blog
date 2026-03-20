# How to Change the Default Port in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Configuration, Ports, Networking, Docker

Description: A guide to changing Portainer's default HTTP and HTTPS ports for security hardening or to avoid conflicts with other services.

## Overview

Portainer listens on port 9443 (HTTPS) and 8000 (Edge agent tunnel) by default. You may need to change these ports to avoid conflicts with other services, comply with security policies, or put Portainer on a non-standard port to reduce attack surface. This guide covers changing ports for Docker standalone, Docker Swarm, and Kubernetes deployments.

## Prerequisites

- Running Portainer installation
- Docker CLI access
- Firewall access to open new ports

## Default Portainer Ports

| Port | Protocol | Purpose |
|---|---|---|
| 9443 | HTTPS | Web UI and API |
| 9000 | HTTP | Web UI (legacy, disabled by default in 2.x) |
| 8000 | TCP | Edge agent tunnel server |

## Method 1: Change Ports (Docker Standalone)

```bash
# Stop and remove existing container

docker stop portainer
docker rm portainer

# Redeploy with custom ports
# Example: Move HTTPS to 443, Edge to 8001
docker run -d \
  -p 443:9443 \
  -p 8001:8000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Verify
docker ps | grep portainer
```

## Method 2: Run Portainer on Custom Internal Port

To change the internal port Portainer binds to (not just the host mapping):

```bash
docker run -d \
  -p 7443:7443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --http-disabled \
  --ssl \
  --sslcert /data/certs/portainer.crt \
  --sslkey /data/certs/portainer.key \
  --bind :7443
```

## Method 3: Change Ports in Docker Compose

```yaml
# docker-compose.yml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "443:9443"      # Host 443 → Container 9443
      - "8001:8000"     # Custom Edge port
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

```bash
# Apply the change
docker compose down
docker compose up -d
```

## Method 4: Change Ports in Docker Swarm

```bash
# Update the service
docker service update \
  --publish-rm 9443 \
  --publish-add published=443,target=9443 \
  portainer_portainer
```

## Method 5: Change Ports in Kubernetes (Helm)

```bash
# Create custom values
cat > portainer-values.yaml << 'EOF'
service:
  type: LoadBalancer
  httpsPort: 443
  httpEnabled: false
  edgePort: 8001
EOF

helm upgrade portainer portainer/portainer \
  --namespace portainer \
  -f portainer-values.yaml
```

## Step: Update Firewall Rules

```bash
# After changing ports, update firewall

# Ubuntu/Debian (ufw)
sudo ufw allow 443/tcp
sudo ufw allow 8001/tcp
sudo ufw delete allow 9443/tcp
sudo ufw reload

# RHEL/CentOS/Rocky (firewalld)
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=8001/tcp
sudo firewall-cmd --permanent --remove-port=9443/tcp
sudo firewall-cmd --reload
```

## Step: Update Edge Agent Configuration

If you changed the Edge tunnel port (8000), update any connected Edge agents:

```bash
# Re-deploy Edge agent with new tunnel URL
docker run -d \
  --name portainer_edge_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  -e EDGE=1 \
  -e EDGE_ID=<your-edge-id> \
  -e EDGE_KEY=<your-edge-key> \
  -e EDGE_INSECURE_POLL=1 \
  portainer/agent:latest \
  -e EDGE_TUNNEL_SERVER_ADDR=portainer.example.com:8001
```

## Conclusion

Changing Portainer's default ports is straightforward for Docker standalone deployments - simply remove and redeploy the container with different port mappings. For Swarm and Kubernetes, use the respective service/Helm update mechanisms. Always update firewall rules and Edge agent configurations after changing ports to ensure continued connectivity.
