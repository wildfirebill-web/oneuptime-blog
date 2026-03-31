# How to Configure RKE2 Registry Mirror

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Registry Mirror, Container Image, Performance, Rancher

Description: Learn how to configure registry mirrors in RKE2 to redirect container image pulls to local mirrors, improving performance and reducing external bandwidth.

Registry mirrors allow you to redirect container image pulls from public registries to local or regional mirrors. This reduces external bandwidth usage, improves pull performance, and adds resilience when public registries are slow or unavailable. This guide covers configuring registry mirrors in RKE2 using the `registries.yaml` file.

## Prerequisites

- RKE2 installed on your nodes
- A registry mirror service (Harbor, Nexus, Sonatype, or Docker Hub mirror)
- Network access from nodes to the mirror
- Understanding of container registry concepts

## What is a Registry Mirror?

A registry mirror is a copy or proxy of a container registry. Instead of pulling images directly from Docker Hub or other public registries, containerd checks the mirror first. If the image is found in the mirror, it's pulled locally (fast). If not, the mirror fetches it from the upstream registry and caches it.

## Step 1: Configure Docker Hub Mirror

```yaml
# /etc/rancher/rke2/registries.yaml - Docker Hub mirror

mirrors:
  # Mirror Docker Hub pulls
  "docker.io":
    endpoint:
    # List mirrors in priority order (first available is used)
    - "https://registry-mirror.example.com"
    - "https://mirror.gcr.io"          # Google's Docker Hub mirror
    - "https://registry-1.docker.io"   # Fallback to original
```

## Step 2: Configure Multiple Registry Mirrors

```yaml
# /etc/rancher/rke2/registries.yaml - Multiple mirrors
mirrors:
  # Docker Hub mirror
  "docker.io":
    endpoint:
    - "https://dockerhub-mirror.internal.example.com"

  # GitHub Container Registry mirror
  "ghcr.io":
    endpoint:
    - "https://ghcr-mirror.internal.example.com"

  # Google Container Registry mirror
  "gcr.io":
    endpoint:
    - "https://gcr-mirror.internal.example.com"

  # Kubernetes registry mirror (quay.io)
  "quay.io":
    endpoint:
    - "https://quay-mirror.internal.example.com"

  # k8s.gcr.io / registry.k8s.io mirror
  "registry.k8s.io":
    endpoint:
    - "https://k8s-mirror.internal.example.com"

configs:
  # Authentication for the mirror
  "dockerhub-mirror.internal.example.com":
    auth:
      username: "mirror-reader"
      password: "mirror-password"
    tls:
      ca_file: "/etc/ssl/certs/internal-ca.crt"
```

## Step 3: Set Up a Harbor Registry Mirror

Harbor can act as a proxy cache for public registries:

### Configure Harbor as a Docker Hub Proxy

```bash
# In Harbor UI:
# 1. Go to Registries -> New Endpoint
# Name: dockerhub-proxy
# Type: Docker Hub
# URL: https://hub.docker.com

# 2. Create a proxy project:
# Projects -> New Project
# Project Name: dockerhub
# Enable "Proxy Cache"
# Select "dockerhub-proxy" as the cache registry

# The proxy project URL will be:
# harbor.example.com/dockerhub/<image>

# 3. Configure RKE2 to use the proxy project
```

```yaml
# /etc/rancher/rke2/registries.yaml - Harbor proxy cache
mirrors:
  "docker.io":
    endpoint:
    - "https://harbor.example.com/v2/dockerhub"

configs:
  "harbor.example.com":
    auth:
      username: "robot$rke2-reader"
      password: "harbor-token"
    tls:
      ca_file: "/etc/ssl/certs/harbor-ca.crt"
```

## Step 4: Configure Mirrors with Rewrites

Some mirrors require URL path rewrites:

```yaml
# /etc/rancher/rke2/registries.yaml - Mirror with rewrites
mirrors:
  "registry.k8s.io":
    endpoint:
    - "https://k8s-mirror.example.com"
    rewrite:
      # Rewrite the image path for the mirror
      "^(.*)$": "k8s/$1"

  "gcr.io":
    endpoint:
    - "https://internal-registry.example.com"
    rewrite:
      # Rewrite gcr.io/google_containers -> internal-registry.example.com/google_containers
      "^google_containers/(.*)$": "google/$1"
```

## Step 5: Testing the Mirror Configuration

```bash
# Apply the registries.yaml configuration
sudo cp registries.yaml /etc/rancher/rke2/registries.yaml

# Restart RKE2 to apply
sudo systemctl restart rke2-server   # or rke2-agent for workers

# Test pulling through the mirror
kubectl run mirror-test \
  --image=nginx:latest \
  --rm -it \
  --restart=Never \
  -- sh -c "echo 'Mirror working'"

# Check which images were pulled
sudo /var/lib/rancher/rke2/bin/crictl images | grep nginx

# Check containerd logs to confirm mirror usage
sudo journalctl -u rke2-server | grep -i "mirror" | tail -10

# Verify the image was served from the mirror
# Look for the mirror hostname in the pull logs
sudo journalctl -u rke2-server | grep -E "pulling|pulled" | tail -5
```

## Step 6: Configure Mirrors for Air-Gapped Operation

In air-gapped environments, all images must come from the mirror:

```yaml
# /etc/rancher/rke2/registries.yaml - Air-gapped mirror configuration
mirrors:
  # Route ALL pulls through the internal registry
  "docker.io":
    endpoint:
    - "https://registry.internal.example.com"

  "ghcr.io":
    endpoint:
    - "https://registry.internal.example.com"

  "gcr.io":
    endpoint:
    - "https://registry.internal.example.com"

  "quay.io":
    endpoint:
    - "https://registry.internal.example.com"

  "registry.k8s.io":
    endpoint:
    - "https://registry.internal.example.com"

configs:
  "registry.internal.example.com":
    auth:
      username: "puller"
      password: "secure-password"
    tls:
      # Use the internal CA certificate
      ca_file: "/etc/ssl/certs/internal-ca.crt"
```

## Step 7: Monitor Mirror Cache Hit Rate

```bash
# Check Harbor cache statistics (in Harbor UI or API)
curl -u admin:password \
  "https://harbor.example.com/api/v2.0/statistics" | \
  python3 -m json.tool

# Check containerd image pull statistics
# Using CRI stats
sudo /var/lib/rancher/rke2/bin/crictl stats 2>/dev/null | head -20

# Check node events for image pull sources
kubectl get events -A | grep -E "Pulling|Pulled" | \
  awk '{print $7}' | sort | uniq -c
```

## Conclusion

Registry mirrors dramatically improve image pull performance in Kubernetes clusters, especially for large deployments where many nodes may pull the same images simultaneously. By directing all pulls through a centralized mirror, you also gain visibility into which images are being used across your cluster and can enforce image policies centrally. For production RKE2 deployments, configuring at least one registry mirror for Docker Hub is strongly recommended, as Docker Hub's rate limiting can cause pod scheduling failures on heavily-loaded clusters.
