# How to Use Podman Desktop in Restricted Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Podman Desktop, Security, Enterprise, Restricted Environments

Description: Learn how to configure and use Podman Desktop in restricted corporate or air-gapped environments where network access and permissions are limited.

---

> Podman Desktop works in restricted environments where Docker cannot, thanks to its rootless architecture and flexible configuration options for proxies, registries, and offline operation.

Many enterprise environments impose restrictions on network access, software installation, and system permissions. Podman Desktop is well-suited for these environments because it runs rootless by default and supports offline workflows, proxy configurations, and custom registry mirrors. This guide covers the strategies and configurations needed to use Podman Desktop effectively behind firewalls and in locked-down systems.

---

## Understanding Restricted Environment Challenges

Restricted environments typically present these challenges:

- No root or sudo access on developer machines
- Network traffic routed through proxies
- Limited or no internet access (air-gapped)
- Custom certificate authorities for TLS inspection
- Restricted container registries
- Disk and memory quotas

```bash
# Check your current user permissions
id
groups

# Verify you do not need root for Podman
podman info --format '{{.Host.Security.Rootless}}'
# Should output: true
```

## Running Podman Rootless

Podman runs without root privileges by default, which is ideal for restricted environments:

```bash
# Verify rootless mode
podman info --format '{{.Host.Security.Rootless}}'

# Check the storage location (user-level, no root needed)
podman info --format '{{.Store.GraphRoot}}'

# Run a container without any elevated privileges
podman run --rm alpine echo "Running rootless"

# Check user namespace mappings
podman unshare cat /proc/self/uid_map
```

## Configuring Offline Image Access

In air-gapped environments, pre-load images from external media:

```bash
# On a machine with internet access, save required images
podman pull nginx:alpine
podman pull postgres:16-alpine
podman pull node:18-alpine
podman save -m -o offline-images.tar \
  nginx:alpine postgres:16-alpine node:18-alpine

# Transfer offline-images.tar to the restricted machine via USB or file share

# On the restricted machine, load the images
podman load -i offline-images.tar

# Verify the images are available
podman images
```

## Setting Up a Local Registry Mirror

Run a local registry to serve images within your restricted network:

```bash
# Save the registry image on an internet-connected machine
podman save registry:2 -o registry.tar

# Load it on the restricted network
podman load -i registry.tar

# Start the local registry
podman run -d \
  --name local-registry \
  -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2

# Push pre-loaded images to the local registry
podman tag nginx:alpine localhost:5000/nginx:alpine
podman push localhost:5000/nginx:alpine

# Configure Podman to use the local registry
cat > ~/.config/containers/registries.conf << 'EOF'
unqualified-search-registries = ["localhost:5000"]

[[registry]]
location = "localhost:5000"
insecure = true
EOF
```

## Configuring Proxy Settings

Set up proxy access for environments that route through corporate proxies:

```bash
# Set proxy environment variables for Podman
mkdir -p ~/.config/containers
cat > ~/.config/containers/containers.conf << 'EOF'
[engine]
# HTTP proxy for container operations
http_proxy = "http://proxy.corporate.com:8080"
https_proxy = "http://proxy.corporate.com:8080"
no_proxy = "localhost,127.0.0.1,.corporate.com,10.0.0.0/8"

[engine.env]
http_proxy = "http://proxy.corporate.com:8080"
https_proxy = "http://proxy.corporate.com:8080"
no_proxy = "localhost,127.0.0.1,.corporate.com"
EOF

# Set environment variables for image pulls
export HTTP_PROXY="http://proxy.corporate.com:8080"
export HTTPS_PROXY="http://proxy.corporate.com:8080"
export NO_PROXY="localhost,127.0.0.1,.corporate.com"

# Test connectivity through the proxy
podman pull alpine
```

## Installing Custom CA Certificates

For TLS-inspecting proxies that use custom certificate authorities:

```bash
# Copy your corporate CA certificate
mkdir -p ~/.config/containers/certs.d/registry.corporate.com
cp /path/to/corporate-ca.crt \
  ~/.config/containers/certs.d/registry.corporate.com/ca.crt

# For system-wide trust on Linux
sudo cp /path/to/corporate-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# For macOS, add to the system keychain
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain /path/to/corporate-ca.crt
```

## Managing Storage Quotas

When disk space is limited, configure storage carefully:

```bash
# Check current storage usage
podman system df

# Set storage limits in storage.conf
mkdir -p ~/.config/containers
cat > ~/.config/containers/storage.conf << 'EOF'
[storage]
driver = "overlay"
graphroot = "/home/user/.local/share/containers/storage"

[storage.options.overlay]
# Use fuse-overlayfs for rootless
mount_program = "/usr/bin/fuse-overlayfs"
EOF

# Regular cleanup to stay within quotas
podman system prune -af --volumes

# Remove old images aggressively
podman image prune -af
```

## Running Without Internet Access

For fully air-gapped operation:

```bash
# Disable all external registry access
cat > ~/.config/containers/registries.conf << 'EOF'
unqualified-search-registries = ["localhost:5000"]

# Block all external registries
[[registry]]
location = "docker.io"
blocked = true

[[registry]]
location = "quay.io"
blocked = true

# Only allow your internal registry
[[registry]]
location = "localhost:5000"
insecure = true
EOF

# All pulls will now only work from localhost:5000
podman pull localhost:5000/nginx:alpine
```

## Summary

Podman Desktop excels in restricted environments thanks to its rootless architecture and flexible configuration. By pre-loading images for air-gapped operation, configuring proxies for corporate networks, and setting up local registry mirrors, you can maintain a productive container workflow even with significant restrictions. Custom CA certificate support and storage management round out the capabilities needed for enterprise deployment.
