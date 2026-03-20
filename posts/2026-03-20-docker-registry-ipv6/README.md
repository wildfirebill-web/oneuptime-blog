# How to Configure Docker Registry Access over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Registry, Container Registry, Image Pull

Description: Configure Docker to pull and push images from container registries over IPv6, run a private Docker registry accessible via IPv6, and troubleshoot registry connectivity over IPv6 connections.

## Introduction

Docker can pull and push images over IPv6 when the registry host resolves to an IPv6 address or when you specify an IPv6 address directly. Running a private registry on an IPv6 address requires binding the registry to an IPv6 interface. Docker's daemon needs IPv6 enabled for proper IPv6 registry communication, and the registry address in image names can include IPv6 addresses using bracket notation.

## Pull Images from IPv6 Registry

```bash
# Pull from a registry accessible via IPv6
# Registry at 2001:db8::1 on port 5000
docker pull [2001:db8::1]:5000/myapp:latest

# Pull from a registry with a hostname that resolves to IPv6
docker pull registry.example.com:5000/myapp:latest
# (assumes registry.example.com has AAAA record)

# Verify which IP Docker connected to
docker system info | grep "Registry"

# Force IPv6 DNS resolution for a hostname
# /etc/docker/daemon.json
# { "dns": ["2001:4860:4860::8888", "8.8.8.8"] }
```

## Run a Private Registry Bound to IPv6

```bash
# Run Docker Registry v2 listening on IPv6
docker run -d \
    --name registry \
    -p "[::]:5000:5000" \
    -v /data/registry:/var/lib/registry \
    -e REGISTRY_HTTP_ADDR="[::]:5000" \
    registry:2

# Verify registry is listening on IPv6
ss -tlnp6 | grep 5000

# Test registry access over IPv6
curl -6 http://[::1]:5000/v2/
# {"errors":[{"code":"UNAUTHORIZED"...}]}  or {}

# Push an image to local IPv6 registry
docker tag nginx:latest [::1]:5000/mynginx:latest
docker push [::1]:5000/mynginx:latest

# Pull from local IPv6 registry
docker pull [::1]:5000/mynginx:latest
```

## Registry with TLS and IPv6

```yaml
# docker-compose.yaml for private registry with TLS

services:
  registry:
    image: registry:2
    ports:
      - "[::]:443:443"
    environment:
      REGISTRY_HTTP_ADDR: "[::]:443"
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /var/lib/registry
    volumes:
      - ./certs:/certs
      - registry-data:/var/lib/registry

volumes:
  registry-data:
```

```bash
# Generate self-signed cert for IPv6 registry
# The cert must include the IPv6 address as a SAN
openssl req -newkey rsa:4096 -nodes \
    -keyout registry.key -x509 \
    -days 365 \
    -out registry.crt \
    -subj "/CN=registry.internal" \
    -addext "subjectAltName=IP:2001:db8::1,DNS:registry.internal"

# Trust the cert on Docker clients
sudo mkdir -p /etc/docker/certs.d/[2001:db8::1]:443
sudo cp registry.crt /etc/docker/certs.d/[2001:db8::1]:443/ca.crt
```

## Configure insecure-registries for IPv6

```json
// /etc/docker/daemon.json — allow insecure HTTP registry over IPv6
{
  "ipv6": true,
  "ip6tables": true,
  "insecure-registries": [
    "[::1]:5000",
    "[2001:db8::1]:5000",
    "[fd00:registry::1]:5000"
  ]
}
```

```bash
sudo systemctl restart docker

# Verify insecure registries
docker info | grep -A5 "Insecure"

# Now you can push/pull from the IPv6 registry without TLS
docker pull [2001:db8::1]:5000/myimage:latest
```

## Conclusion

Docker registry access over IPv6 uses bracket notation for IPv6 addresses in image names, e.g., `[2001:db8::1]:5000/image:tag`. Run a private registry bound to IPv6 by setting `REGISTRY_HTTP_ADDR=[::]:5000`. Configure `insecure-registries` in `daemon.json` for HTTP-only IPv6 registries. For TLS-secured registries, generate certificates with IPv6 Subject Alternative Names and trust the CA in `/etc/docker/certs.d/[ipv6]:port/`. Ensure the Docker daemon has IPv6 enabled to support IPv6 registry connections.
