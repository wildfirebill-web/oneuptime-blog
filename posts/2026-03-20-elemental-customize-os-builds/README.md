# How to Customize Elemental OS Builds

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, OS Customization, Docker, Kubernetes, Edge

Description: Extend the base Elemental OS image with custom packages, configurations, and services to meet specific deployment requirements.

## Introduction

Elemental OS images are standard OCI container images, which means you can extend them using standard Dockerfile syntax. This flexibility allows you to install custom software, add configuration files, integrate monitoring agents, and configure system services - all while maintaining the immutable, reproducible properties that make Elemental reliable at scale.

## Customization Strategies

- **Dockerfile extensions**: Add packages, files, and services
- **Multi-stage builds**: Keep image size minimal
- **Build-time secrets**: Handle sensitive configuration securely
- **Version tagging**: Track which build is deployed where

## Creating a Custom Elemental Image

### Basic Package Installation

```dockerfile
# Dockerfile.elemental-custom

FROM registry.suse.com/rancher/sle-micro:latest

# Install monitoring and operations tools
RUN zypper --non-interactive install \
    vim \
    curl \
    jq \
    htop \
    prometheus-node-exporter \
    && zypper clean -a

# Enable node exporter on boot
RUN systemctl enable prometheus-node-exporter
```

### Adding Custom Systemd Services

```dockerfile
FROM registry.suse.com/rancher/sle-micro:latest

# Copy custom service files
COPY systemd/my-agent.service /etc/systemd/system/
COPY scripts/my-agent.sh /usr/local/bin/

# Make script executable
RUN chmod +x /usr/local/bin/my-agent.sh

# Enable the service
RUN systemctl enable my-agent.service
```

### Multi-Stage Build for Size Optimization

```dockerfile
# Build stage: compile custom tools
FROM golang:1.21 AS builder
WORKDIR /app
COPY my-tool/ .
RUN go build -o /usr/local/bin/my-tool ./cmd/my-tool

# Final image: only runtime artifacts
FROM registry.suse.com/rancher/sle-micro:latest

# Copy compiled binary from builder
COPY --from=builder /usr/local/bin/my-tool /usr/local/bin/my-tool

# Copy service definition
COPY my-tool.service /etc/systemd/system/
RUN chmod +x /usr/local/bin/my-tool && \
    systemctl enable my-tool.service
```

## Adding Custom Configuration Files

```dockerfile
FROM registry.suse.com/rancher/sle-micro:latest

# Sysctl tuning for Kubernetes nodes
COPY sysctl-kubernetes.conf /etc/sysctl.d/99-kubernetes.conf

# Custom NTP configuration
COPY chrony.conf /etc/chrony.conf

# Custom SSH daemon config
COPY sshd-custom.conf /etc/ssh/sshd_config.d/10-custom.conf

# Firewall rules
COPY firewall-rules.sh /usr/local/bin/firewall-setup.sh
RUN chmod +x /usr/local/bin/firewall-setup.sh

# Apply configurations
RUN sysctl --system || true
```

## Integrating Monitoring Agents

```dockerfile
FROM registry.suse.com/rancher/sle-micro:latest

# Install Datadog agent
ARG DD_AGENT_VERSION=7.50.0
RUN curl -fsSL https://s3.amazonaws.com/dd-agent/packages/datadog-agent-${DD_AGENT_VERSION}.x86_64.rpm \
    -o /tmp/dd-agent.rpm && \
    rpm -i /tmp/dd-agent.rpm && \
    rm /tmp/dd-agent.rpm

# Copy Datadog config
COPY datadog.yaml /etc/datadog-agent/datadog.yaml

# Enable Datadog
RUN systemctl enable datadog-agent
```

## Building and Publishing

```bash
# Build with version tag
docker build \
  -t my-registry.example.com/elemental-os:v1.2.0 \
  -f Dockerfile.elemental-custom \
  .

# Run basic verification
docker run --rm my-registry.example.com/elemental-os:v1.2.0 \
  bash -c "prometheus-node-exporter --version && my-tool --version"

# Push to registry
docker push my-registry.example.com/elemental-os:v1.2.0

# Create a 'latest' tag
docker tag my-registry.example.com/elemental-os:v1.2.0 \
           my-registry.example.com/elemental-os:latest
docker push my-registry.example.com/elemental-os:latest
```

## CI/CD Pipeline for OS Builds

```yaml
# .github/workflows/elemental-os-build.yaml
name: Build Elemental OS Image

on:
  push:
    branches: [main]
    paths:
      - 'Dockerfile.elemental-custom'
      - 'config/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build OS image
        run: |
          VERSION=$(git describe --tags --always)
          docker build \
            -t my-registry.example.com/elemental-os:${VERSION} \
            -f Dockerfile.elemental-custom \
            .

      - name: Push to registry
        run: |
          VERSION=$(git describe --tags --always)
          docker push my-registry.example.com/elemental-os:${VERSION}
```

## Conclusion

Customizing Elemental OS images using standard Dockerfile patterns gives you full flexibility to build the exact OS environment your nodes need. By treating OS images like application containers, you benefit from versioning, CI/CD pipelines, and registry management. This approach ensures all nodes run the exact same, tested OS image, eliminating configuration drift and making upgrades predictable and safe.
