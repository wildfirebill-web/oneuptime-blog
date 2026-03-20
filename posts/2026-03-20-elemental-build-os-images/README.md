# How to Build Elemental OS Images

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, OS Images, Edge, Rancher

Description: A comprehensive guide to building custom Elemental OS images using the elemental-toolkit for deploying to bare metal and edge nodes.

## Introduction

Elemental OS images are immutable, container-based operating system images built on top of SLE Micro or openSUSE MicroOS. These images are designed for edge and bare metal deployments where consistency and reproducibility are critical. The elemental-toolkit provides the tooling to build, customize, and publish OS images.

## Prerequisites

- Docker or Podman installed
- Access to a container registry
- `elemental` CLI installed (or use the container image)

## Installing the Elemental CLI

```bash
# Install via package manager (openSUSE/SUSE)

zypper install elemental

# Or run as a container
docker run --privileged --rm -it \
  registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest \
  --help
```

## Understanding Elemental Image Layers

Elemental OS images are built as OCI container images with a specific layer structure:

```text
Base OS Layer (SLE Micro / openSUSE MicroOS)
    └── System Extensions (packages, configs)
        └── Elemental Agent Layer
            └── Custom Packages/Configuration
```

## Building a Basic OS Image

### Create a Dockerfile

```dockerfile
# Dockerfile.elemental
# Start from the official Elemental base image
FROM registry.suse.com/rancher/sle-micro:latest

# Install additional packages
RUN zypper --non-interactive install \
    vim \
    curl \
    htop \
    && zypper clean -a

# Copy custom configuration files
COPY custom-config/ /etc/

# Install custom monitoring agent
RUN curl -fsSL https://get.custom-agent.io | sh

# Set OS release information
ARG IMAGE_TAG=latest
RUN echo "IMAGE_TAG=${IMAGE_TAG}" >> /etc/os-release
```

```bash
# Build the container image
docker build \
  -t my-registry.example.com/elemental-os:v1.0.0 \
  -f Dockerfile.elemental \
  .

# Push to registry
docker push my-registry.example.com/elemental-os:v1.0.0
```

## Building with the ManagedOSVersionChannel

Define a channel to track OS versions:

```yaml
# os-version-channel.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: ManagedOSVersionChannel
metadata:
  name: my-os-channel
  namespace: fleet-default
spec:
  # Sync interval
  syncInterval: "1h"
  type: custom
  options:
    image: "my-registry.example.com/elemental-os-channel:latest"
```

## Creating a ManagedOSVersion

```yaml
# managed-os-version.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: ManagedOSVersion
metadata:
  name: elemental-v1.0.0
  namespace: fleet-default
spec:
  type: container
  version: "v1.0.0"
  minVersion: "0.0.0"
  # The OS container image
  metadata:
    upgradeImage: "my-registry.example.com/elemental-os:v1.0.0"
    # Cloud-config applied during upgrade
    cloudConfig: ""
```

## Customizing the OS Build

### Adding Custom Packages

```dockerfile
# Install enterprise monitoring stack
FROM registry.suse.com/rancher/sle-micro:latest

# Add custom repository
RUN zypper addrepo https://packages.example.com/repo my-repo

# Install monitoring packages
RUN zypper --non-interactive --gpg-auto-import-keys install \
    prometheus-node-exporter \
    filebeat \
    && zypper clean -a

# Configure node exporter to start on boot
RUN systemctl enable prometheus-node-exporter
```

### Adding Custom Systemd Services

```dockerfile
FROM registry.suse.com/rancher/sle-micro:latest

# Copy custom systemd unit
COPY my-service.service /etc/systemd/system/

# Enable the service
RUN systemctl enable my-service
```

## Building ISO Images

To create bootable ISO images for initial provisioning:

```bash
# Use the elemental CLI to build an ISO
docker run --privileged --rm -v $(pwd):/workspace \
  registry.suse.com/rancher/elemental-toolkit/elemental-cli:latest \
  build-iso \
  --name elemental-custom \
  --local \
  my-registry.example.com/elemental-os:v1.0.0

# The ISO will be created in the current directory
ls -la elemental-custom.iso
```

## Validating the Image

```bash
# Inspect the image layers
docker inspect my-registry.example.com/elemental-os:v1.0.0

# Run a quick sanity check
docker run --rm my-registry.example.com/elemental-os:v1.0.0 \
  cat /etc/os-release
```

## Conclusion

Building custom Elemental OS images gives you complete control over the software stack deployed to your edge and bare metal nodes. By combining OCI container tooling with the Elemental operator's declarative model, you can version, test, and roll out OS changes with the same practices used for application containers.
