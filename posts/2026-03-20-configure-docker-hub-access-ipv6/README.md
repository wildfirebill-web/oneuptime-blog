# How to Configure Docker Hub Access over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Container Registry, Docker Hub, Networking, DevOps

Description: Configure Docker to pull and push images from Docker Hub over IPv6, including Docker daemon IPv6 settings, DNS configuration, and troubleshooting connectivity issues.

---

Docker Hub supports IPv6 access, and configuring Docker to use IPv6 for registry communication can improve performance and security in IPv6-enabled infrastructure. This requires configuring the Docker daemon and ensuring proper DNS resolution.

## Enabling IPv6 in Docker Daemon

Docker's IPv6 support must be explicitly enabled:

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80",
  "ip6tables": true,
  "experimental": false,
  "dns": ["2001:4860:4860::8888", "2001:4860:4860::8844", "8.8.8.8"]
}
```

```bash
# Reload Docker daemon

sudo systemctl daemon-reload
sudo systemctl restart docker

# Verify IPv6 is enabled
docker network inspect bridge | grep -i ipv6
docker run --rm alpine ip -6 addr show
```

## Verifying Docker Hub IPv6 Connectivity

```bash
# Check if Docker Hub resolves to IPv6
dig AAAA registry-1.docker.io +short
dig AAAA auth.docker.io +short
dig AAAA hub.docker.com +short

# Test IPv6 reachability to Docker Hub
curl -6 -v https://registry-1.docker.io/v2/ 2>&1 | head -20

# Check if Docker can connect
docker info | grep -i "registry\|ipv6"
```

## Pulling Images from Docker Hub over IPv6

Once Docker daemon is configured with IPv6:

```bash
# Pull an image - Docker will use IPv6 if available
docker pull nginx:latest

# Force a fresh pull to test connectivity
docker pull --no-cache alpine:3.18

# Verify which network path was used (check system logs)
sudo journalctl -u docker | grep "registry-1.docker.io"

# Test with explicit network configuration
docker run --rm --network bridge6 alpine ping6 -c 3 2001:4860:4860::8888
```

## Creating an IPv6-Enabled Docker Network

For containers that need IPv6 access to registries:

```bash
# Create a Docker network with IPv6 support
docker network create \
  --ipv6 \
  --subnet "fd00:10::/80" \
  --gateway "fd00:10::1" \
  myipv6net

# Verify the network
docker network inspect myipv6net | grep -A 5 "IPv6"

# Run a container on the IPv6 network
docker run --rm --network myipv6net \
  alpine curl -6 https://hub.docker.com
```

## Configuring Docker Behind an IPv6 Proxy

If accessing Docker Hub through an IPv6 proxy:

```bash
# Configure Docker to use the proxy
sudo mkdir -p /etc/systemd/system/docker.service.d

cat > /etc/systemd/system/docker.service.d/proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://[2001:db8::proxy]:3128"
Environment="HTTPS_PROXY=http://[2001:db8::proxy]:3128"
Environment="NO_PROXY=localhost,::1,127.0.0.1,fd00::/8"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Docker Compose with IPv6 Networks

```yaml
# docker-compose.yml with IPv6 support
version: '3.8'

networks:
  app_net:
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: "2001:db8:app::/80"
        - subnet: "172.20.0.0/16"

services:
  webapp:
    image: nginx:latest
    networks:
      app_net:
        ipv6_address: "2001:db8:app::2"
    ports:
      - "80:80"
      - "[::]:80:80"  # IPv6 port binding
```

## Troubleshooting Docker Hub IPv6 Access

```bash
# Check if the Docker daemon can reach Docker Hub
docker run --rm curlimages/curl \
  curl -6 https://registry-1.docker.io/v2/

# Test DNS resolution from within Docker
docker run --rm alpine nslookup -type=AAAA registry-1.docker.io

# Check Docker network firewall rules for IPv6
sudo ip6tables -L FORWARD -n | grep "docker\|172\|fd00"

# Enable IPv6 forwarding if needed
sudo sysctl -w net.ipv6.conf.all.forwarding=1
echo "net.ipv6.conf.all.forwarding = 1" | sudo tee -a /etc/sysctl.conf
```

Enabling Docker's IPv6 support and configuring the daemon to use IPv6-capable DNS servers allows seamless image pulls from Docker Hub over IPv6, improving network efficiency in modern dual-stack infrastructure.
