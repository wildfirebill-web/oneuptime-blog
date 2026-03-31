# How to Push and Pull Container Images over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, IPv6, Container Image, Registry, Podman, Containerd, DevOps

Description: Push and pull container images over IPv6 using Docker, Podman, and containerd, covering registry authentication, IPv6 address syntax, and common runtime configurations.

---

Pushing and pulling container images over IPv6 requires proper Docker daemon configuration, correct registry URL formatting, and ensuring the container runtime can resolve and connect to registry endpoints via IPv6.

## Prerequisites: Enable Docker IPv6

```json
// /etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80",
  "ip6tables": true,
  "dns": ["2001:4860:4860::8888", "8.8.8.8"]
}
```

```bash
sudo systemctl restart docker
```

## Pulling Images Using IPv6 Registry Address

When using a registry that only has an IPv6 address (or you want to force IPv6):

```bash
# Pull from a registry at an IPv6 address

# Use bracket notation for IPv6 in the registry URL
docker pull [2001:db8::1]:5000/myapp:latest

# Pull from a hostname with AAAA record
docker pull registry.example.com/myapp:latest
# (Docker uses the AAAA record automatically)

# Pull from Docker Hub (resolves to IPv6 if available)
docker pull nginx:latest

# Verify which address was used
sudo tcpdump -i eth0 -n ip6 and tcp port 443 &
docker pull alpine:latest
sudo kill %1
```

## Pushing Images to an IPv6 Registry

```bash
# Tag image for IPv6 registry
docker tag myapp:latest [2001:db8::1]:5000/myapp:latest

# Push to IPv6-addressed registry
docker push [2001:db8::1]:5000/myapp:latest

# Push to hostname-based registry (AAAA record required)
docker tag myapp:latest registry.example.com/myapp:latest
docker push registry.example.com/myapp:latest

# Push to Docker Hub over IPv6
docker tag myapp:latest username/myapp:latest
docker push username/myapp:latest
```

## Running a Local IPv6 Registry

For testing, run a local registry on IPv6:

```bash
# Start a local registry listening on IPv6
docker run -d \
  --name local-registry \
  -p "[::]:5000:5000" \
  -v /data/registry:/var/lib/registry \
  registry:2

# Verify it's listening on IPv6
ss -tlnp | grep :5000

# Configure Docker to trust this insecure registry
# /etc/docker/daemon.json
{
  "insecure-registries": ["[::1]:5000", "[2001:db8::1]:5000"]
}

# Restart Docker
sudo systemctl restart docker

# Push to the local IPv6 registry
docker tag nginx:latest [::1]:5000/nginx:latest
docker push [::1]:5000/nginx:latest

# Pull from it
docker pull [::1]:5000/nginx:latest
```

## Using Podman with IPv6 Registries

Podman doesn't require a daemon and handles IPv6 natively:

```bash
# Pull image over IPv6 with Podman
podman pull registry.example.com/myapp:latest

# Push to IPv6 registry
podman push [2001:db8::1]:5000/myapp:latest

# Configure insecure registries for Podman
cat > /etc/containers/registries.conf.d/ipv6-registry.conf << 'EOF'
[[registry]]
location = "[2001:db8::1]:5000"
insecure = true
EOF

# Test connectivity
podman info | grep registries
```

## Using containerd with IPv6 Registries

For containerd (used by Kubernetes):

```toml
# /etc/containerd/config.toml

[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    # Mirror for IPv6 registry
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.example.com"]
      endpoint = ["https://registry.example.com"]

  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."[2001:db8::1]:5000"]
      [plugins."io.containerd.grpc.v1.cri".registry.configs."[2001:db8::1]:5000".tls]
        insecure_skip_verify = true
```

```bash
sudo systemctl restart containerd

# Pull using ctr
ctr images pull registry.example.com/myapp:latest
```

## Multi-Platform Image Push over IPv6

```bash
# Create and push multi-platform manifest over IPv6
docker buildx create --name ipv6builder --use

# Build for multiple architectures and push
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag registry.example.com/myapp:latest \
  --push \
  .

# Verify the manifest
docker manifest inspect registry.example.com/myapp:latest
```

## Troubleshooting Image Push/Pull over IPv6

```bash
# Error: no such host
# Fix: Add AAAA record or use IP directly
dig AAAA registry.example.com

# Error: x509: certificate
# Fix: Trust the registry CA
sudo cp registry-ca.crt \
  /etc/docker/certs.d/[2001:db8::1]:5000/ca.crt

# Error: dial tcp: connect: network unreachable
# Fix: Check IPv6 routing
ip -6 route show
ping6 -c 3 2001:db8::1
```

Container image operations over IPv6 work seamlessly with properly configured Docker daemons, Podman, or containerd - the main requirement is using correct bracket notation for IPv6 addresses in registry URLs.
