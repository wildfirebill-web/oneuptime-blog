# How to Set Up Registry Mirroring in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Container Registry, Mirroring, Performance

Description: Configure registry mirroring in Rancher to speed up image pulls, reduce external bandwidth, and improve cluster reliability in restricted or air-gapped environments.

## Introduction

Registry mirroring creates a local cache or proxy of a container registry, reducing pull times and external network usage. In Rancher-managed clusters, configuring mirrors ensures that frequently used images are served locally, improving pod startup times and reducing dependency on external registries. This is especially valuable for air-gapped deployments or environments with limited internet bandwidth.

## Prerequisites

- Rancher managing RKE2 or K3s clusters
- A local registry (Harbor, Docker Registry, or Nexus)
- SSH access to cluster nodes (for direct configuration)
- kubectl access to your cluster

## Step 1: Understanding Registry Mirroring

Registry mirroring works as a pull-through cache. When a node requests an image:
1. The node checks the mirror first.
2. If found, the mirror returns the cached image.
3. If not, the mirror fetches from the upstream registry, caches it, and returns it.

## Step 2: Set Up a Local Registry Mirror with Docker Registry

```bash
# Deploy a pull-through cache for Docker Hub using Docker Registry
docker run -d \
  --restart=always \
  --name registry-mirror \
  -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
  -e REGISTRY_PROXY_USERNAME=myusername \
  -e REGISTRY_PROXY_PASSWORD=mypassword \
  -v /data/registry-mirror:/var/lib/registry \
  -p 5000:5000 \
  registry:2
```

## Step 3: Configure Mirroring in RKE2

Create the registries configuration on each RKE2 node:

```yaml
# /etc/rancher/rke2/registries.yaml - RKE2 registry mirror config
mirrors:
  # Mirror for Docker Hub images
  "docker.io":
    endpoints:
      - "http://registry-mirror.internal:5000"
      - "https://registry-1.docker.io"  # Fallback to Docker Hub

  # Mirror for quay.io images
  "quay.io":
    endpoints:
      - "http://registry-mirror.internal:5001"
      - "https://quay.io"  # Fallback

  # Mirror for gcr.io images
  "gcr.io":
    endpoints:
      - "http://registry-mirror.internal:5002"
      - "https://gcr.io"  # Fallback

  # Mirror for k8s.gcr.io (now registry.k8s.io)
  "registry.k8s.io":
    endpoints:
      - "http://registry-mirror.internal:5003"
      - "https://registry.k8s.io"  # Fallback

configs:
  # Credentials for the mirror if authentication is required
  "registry-mirror.internal:5000":
    auth:
      username: mirror-user
      password: mirror-password
    tls:
      # Skip TLS verification for internal mirror (not recommended in production)
      insecureSkipVerify: true
```

Restart RKE2 to apply changes:

```bash
# Apply on server nodes
sudo systemctl restart rke2-server

# Apply on agent nodes
sudo systemctl restart rke2-agent
```

## Step 4: Configure Mirroring in K3s

```yaml
# /etc/rancher/k3s/registries.yaml - K3s registry mirror config
mirrors:
  "docker.io":
    endpoints:
      - "https://registry-mirror.internal"
      - "https://registry-1.docker.io"
  "ghcr.io":
    endpoints:
      - "https://registry-mirror.internal/ghcr"
      - "https://ghcr.io"
```

```bash
# Restart K3s to apply
sudo systemctl restart k3s
```

## Step 5: Deploy a Harbor Mirror with Rancher Helm

Deploy Harbor as a comprehensive mirror/registry using Helm:

```yaml
# harbor-mirror-values.yaml - Harbor as pull-through cache
expose:
  type: clusterIP

externalURL: https://harbor.internal

# Configure proxy cache projects
proxy:
  httpProxy: ""
  httpsProxy: ""
  noProxy: "127.0.0.1,localhost,.local,.internal"

persistence:
  persistentVolumeClaim:
    registry:
      size: 200Gi  # Size based on expected cached images
```

```bash
helm install harbor harbor/harbor \
  --namespace harbor \
  --values harbor-mirror-values.yaml
```

## Step 6: Create a Proxy Cache Project in Harbor

```bash
# Create a proxy cache for Docker Hub via Harbor API
curl -X POST "https://harbor.internal/api/v2.0/registries" \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "name": "docker-hub-proxy",
    "type": "docker-hub",
    "url": "https://hub.docker.com",
    "description": "Docker Hub proxy cache"
  }'

# Create a proxy project
curl -X POST "https://harbor.internal/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "project_name": "dockerhub",
    "registry_id": 1,
    "public": true,
    "metadata": {
      "auto_scan": "true"
    }
  }'
```

Now configure RKE2 to use Harbor as a mirror:

```yaml
# /etc/rancher/rke2/registries.yaml - Using Harbor as mirror
mirrors:
  "docker.io":
    endpoints:
      - "https://harbor.internal/dockerhub"
configs:
  "harbor.internal":
    tls:
      ca_file: /etc/ssl/certs/harbor-ca.crt
    auth:
      username: robot-puller
      password: <robot-token>
```

## Step 7: Verify Mirror is Working

```bash
# Pull an image and check if it was served from the mirror
crictl pull nginx:latest

# Check RKE2 agent logs for mirror usage
journalctl -u rke2-agent | grep "mirror"

# Check Harbor proxy cache statistics
curl -s "https://harbor.internal/api/v2.0/projects/dockerhub" \
  -u admin:password | jq '.metadata'
```

## Step 8: Configure Automated Warming (Pre-caching)

Pre-populate the mirror with commonly used images:

```bash
#!/bin/bash
# warm-cache.sh - Pre-cache commonly used images in mirror

MIRROR="harbor.internal/dockerhub"
IMAGES=(
  "nginx:1.25"
  "alpine:3.18"
  "busybox:latest"
  "redis:7"
  "postgres:15"
)

for IMAGE in "${IMAGES[@]}"; do
  echo "Warming cache for: $IMAGE"
  docker pull $MIRROR/$IMAGE
done
```

## Troubleshooting

```bash
# Check if containerd is using the mirror
cat /var/lib/rancher/rke2/agent/etc/containerd/config.toml | grep -A 5 mirrors

# Test mirror connectivity
curl -I http://registry-mirror.internal:5000/v2/

# Check containerd logs
journalctl -u rke2-agent -f | grep -i "registry\|mirror\|pull"
```

## Conclusion

Registry mirroring significantly improves cluster reliability and performance by reducing dependency on external registries and cutting down image pull times. For production environments, deploy Harbor as a comprehensive mirror solution that provides additional features like vulnerability scanning and access control. Always configure fallback endpoints to ensure workloads can still pull images even if the mirror is unavailable.
