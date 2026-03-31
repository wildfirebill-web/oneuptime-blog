# How to Build an Image with DNS Configuration with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Container Image, Build, DNS, Network Configuration

Description: Learn how to configure custom DNS settings during Podman image builds for resolving internal hostnames, using private DNS servers, and working in corporate networks.

---

> Custom DNS configuration during builds solves hostname resolution issues in corporate networks and private infrastructure.

Container builds frequently fail because they cannot resolve internal hostnames for private package registries, corporate proxies, or internal services. Podman provides `--dns`, `--dns-search`, and `--add-host` flags to configure DNS resolution during builds. This guide covers all the practical DNS configuration patterns.

---

## Default DNS Behavior

By default, Podman uses the host's DNS configuration from `/etc/resolv.conf` for container builds.

```bash
# Check your host DNS configuration

cat /etc/resolv.conf

# See what DNS the build uses by default
cat > Containerfile << 'EOF'
FROM docker.io/library/alpine:latest
RUN cat /etc/resolv.conf
CMD ["sh"]
EOF

podman build -t dns-test:latest .
```

## Setting Custom DNS Servers

Use the `--dns` flag to specify DNS servers for the build process.

```bash
# Use a specific DNS server
podman build --dns=10.0.0.53 -t myapp:latest .

# Use multiple DNS servers (fallback order)
podman build \
  --dns=10.0.0.53 \
  --dns=10.0.0.54 \
  -t myapp:latest .

# Use Google's public DNS
podman build --dns=8.8.8.8 --dns=8.8.4.4 -t myapp:latest .

# Use Cloudflare's DNS
podman build --dns=1.1.1.1 --dns=1.0.0.1 -t myapp:latest .
```

## Setting DNS Search Domains

Use `--dns-search` to configure search domains, allowing short hostname resolution.

```bash
# Add search domains
podman build \
  --dns-search=corp.example.com \
  --dns-search=internal.example.com \
  -t myapp:latest .
```

With search domains configured, a hostname like `registry` is resolved as `registry.corp.example.com` and `registry.internal.example.com`.

```bash
cat > Containerfile << 'EOF'
FROM docker.io/library/alpine:latest
RUN apk add --no-cache curl

# With dns-search=corp.example.com, this resolves to
# registry.corp.example.com
RUN curl -s http://registry:5000/v2/ || echo "Registry not available"

CMD ["sh"]
EOF

podman build \
  --dns=10.0.0.53 \
  --dns-search=corp.example.com \
  -t myapp:latest .
```

## Adding Static Host Entries

Use `--add-host` to add entries to `/etc/hosts` during the build.

```bash
# Add a single host entry
podman build \
  --add-host=registry.internal:192.168.1.100 \
  -t myapp:latest .

# Add multiple host entries
podman build \
  --add-host=registry.internal:192.168.1.100 \
  --add-host=api.internal:192.168.1.101 \
  --add-host=db.internal:192.168.1.102 \
  -t myapp:latest .
```

```bash
cat > Containerfile << 'EOF'
FROM docker.io/library/python:3.12-slim

WORKDIR /app
COPY requirements.txt .

# Uses the custom host entry for the internal PyPI server
RUN pip install --no-cache-dir \
  --index-url http://pypi.internal:8080/simple \
  --trusted-host pypi.internal \
  -r requirements.txt

COPY . .
CMD ["python", "app.py"]
EOF

podman build \
  --add-host=pypi.internal:192.168.1.50 \
  -t myapp:latest .
```

## Corporate Network Configuration

A complete DNS setup for corporate environments.

```bash
# Corporate build with full DNS configuration
podman build \
  --dns=10.10.0.10 \
  --dns=10.10.0.11 \
  --dns-search=corp.example.com \
  --dns-search=dev.example.com \
  --add-host=npm-registry:10.10.5.20 \
  --add-host=pypi-mirror:10.10.5.21 \
  --add-host=maven-central:10.10.5.22 \
  -t myapp:latest .
```

## DNS Configuration for Different Package Managers

Configure DNS for accessing internal package mirrors.

```bash
# Node.js with internal npm registry
cat > Containerfile << 'EOF'
FROM docker.io/library/node:20-alpine
WORKDIR /app
RUN npm config set registry http://npm-registry.internal:4873/
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
EOF

podman build \
  --dns=10.0.0.53 \
  --dns-search=internal.corp.example.com \
  -t node-app:latest .
```

```bash
# Python with internal PyPI
cat > Containerfile << 'EOF'
FROM docker.io/library/python:3.12-slim
WORKDIR /app
RUN pip config set global.index-url http://pypi-mirror.internal:8080/simple && \
    pip config set global.trusted-host pypi-mirror.internal
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
EOF

podman build \
  --add-host=pypi-mirror.internal:10.0.5.21 \
  -t python-app:latest .
```

## DNS with Host Network

When using `--network=host`, the build uses the host's DNS directly.

```bash
# Host networking uses host DNS automatically
podman build --network=host -t myapp:latest .

# You can still override DNS with host networking
podman build \
  --network=host \
  --dns=10.0.0.53 \
  -t myapp:latest .
```

## Debugging DNS Issues

Troubleshoot DNS resolution problems during builds.

```bash
cat > Containerfile.dns-debug << 'EOF'
FROM docker.io/library/alpine:latest
RUN apk add --no-cache bind-tools curl

# Show DNS configuration
RUN echo "=== /etc/resolv.conf ===" && cat /etc/resolv.conf

# Show /etc/hosts
RUN echo "=== /etc/hosts ===" && cat /etc/hosts

# Test DNS resolution
RUN echo "=== DNS Tests ===" && \
    nslookup google.com || echo "google.com: FAILED" && \
    nslookup registry.internal || echo "registry.internal: FAILED"

# Test connectivity
RUN curl -v --connect-timeout 5 https://registry.npmjs.org/ 2>&1 | head -20 || true

CMD ["sh"]
EOF

# Debug with default DNS
podman build -f Containerfile.dns-debug -t dns-debug:default .

# Debug with custom DNS
podman build -f Containerfile.dns-debug \
  --dns=10.0.0.53 \
  --dns-search=corp.example.com \
  --add-host=registry.internal:192.168.1.100 \
  -t dns-debug:custom .
```

## DNS Options

Set custom DNS options for fine-tuning resolver behavior.

```bash
# Set DNS options
podman build \
  --dns=10.0.0.53 \
  --dns-option=ndots:5 \
  --dns-option=timeout:2 \
  --dns-option=attempts:3 \
  -t myapp:latest .
```

## Build Script with DNS Configuration

Create a reusable build script that applies standard DNS settings.

```bash
#!/bin/bash
# corp-build.sh - Build with corporate DNS settings

set -e

IMAGE="${1:?Usage: $0 IMAGE [TAG]}"
TAG="${2:-latest}"

# Corporate DNS settings
CORP_DNS="--dns=10.10.0.10 --dns=10.10.0.11"
CORP_SEARCH="--dns-search=corp.example.com --dns-search=dev.example.com"
CORP_HOSTS="--add-host=npm-registry:10.10.5.20 --add-host=pypi-mirror:10.10.5.21"

podman build \
  ${CORP_DNS} \
  ${CORP_SEARCH} \
  ${CORP_HOSTS} \
  -t "${IMAGE}:${TAG}" \
  .

echo "Built ${IMAGE}:${TAG} with corporate DNS"
```

```bash
# Usage
chmod +x corp-build.sh
./corp-build.sh myapp v1.0.0
```

## Podman Configuration for Persistent DNS

Set default DNS configuration in Podman's configuration file so you do not need to specify it on every build.

```bash
# Edit containers.conf
# Location: ~/.config/containers/containers.conf (rootless)
# or /etc/containers/containers.conf (root)

cat >> ~/.config/containers/containers.conf << 'EOF'

[containers]
dns_servers = ["10.0.0.53", "10.0.0.54"]
dns_searches = ["corp.example.com"]
EOF

# Now all builds use these DNS settings by default
podman build -t myapp:latest .
```

## Summary

DNS configuration during Podman builds is essential for corporate networks, internal registries, and custom infrastructure. Use `--dns` for custom nameservers, `--dns-search` for search domains, and `--add-host` for static hostname mappings. Debug DNS issues with a dedicated test Containerfile, and set default DNS configuration in `containers.conf` for persistent settings across all builds.
