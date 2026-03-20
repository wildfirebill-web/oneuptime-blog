# How to Handle Docker-Specific Features Not in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Docker, Migration, Compatibility, Workarounds

Description: Learn how to handle Docker-specific features that are not directly available in Podman, including workarounds and alternative approaches.

---

> Not every Docker feature has a direct Podman equivalent, but for each gap there is a practical workaround or a better alternative within the Podman ecosystem.

Podman covers the vast majority of Docker's functionality, but some Docker-specific features are either missing, implemented differently, or replaced by Podman-native alternatives. Knowing these differences upfront prevents frustration during migration. This guide catalogs the most common Docker features that behave differently in Podman and provides practical solutions for each.

---

## Docker Swarm

Docker Swarm is Docker's built-in orchestration system. Podman does not include a Swarm equivalent.

```bash
# Docker Swarm commands (not available in Podman)

# docker swarm init
# docker service create
# docker stack deploy

# Alternative: Use Kubernetes with Podman
# Generate Kubernetes YAML from existing Podman containers
podman generate kube mycontainer > mycontainer.yaml

# Deploy with Kubernetes (k3s, kind, or minikube)
kubectl apply -f mycontainer.yaml

# Alternative: Use Podman pods for multi-container grouping
podman pod create --name myapp -p 8080:80
podman run -d --pod myapp --name web nginx
podman run -d --pod myapp --name api myapi:latest

# Generate a systemd service for auto-restart and management
podman generate systemd --new --name myapp > myapp.service
```

## Docker Compose (Partial Differences)

Docker Compose mostly works with Podman, but some features differ.

```bash
# Features that work identically:
# - services, volumes, networks
# - build, ports, environment
# - depends_on, healthcheck

# Features with differences:
# docker compose watch (live sync) - limited support
# docker compose profiles - supported in podman-compose

# Use podman-compose for best compatibility
pip3 install podman-compose
podman-compose up -d

# Or use docker-compose with the Podman socket
export DOCKER_HOST="unix://${XDG_RUNTIME_DIR}/podman/podman.sock"
systemctl --user start podman.socket
docker-compose up -d
```

## Container Restart Policies

Docker uses the daemon to handle restart policies. Podman uses systemd instead.

```bash
# Docker restart policy
docker run -d --restart=always --name myapp myimage

# Podman: --restart=always works but requires the Podman service
# For reliable restarts, generate and use systemd services
podman run -d --name myapp myimage

# Generate a systemd service
podman generate systemd --new --name myapp \
  --restart-policy=always > ~/.config/systemd/user/myapp.service

# Enable and start the service
systemctl --user daemon-reload
systemctl --user enable --now myapp.service

# The container now restarts on failure and on boot
systemctl --user status myapp.service
```

## Docker Build Cache

Docker uses BuildKit with advanced caching. Podman has its own caching mechanism.

```bash
# Docker BuildKit cache mount
# DOCKER_BUILDKIT=1 docker build .

# Podman supports RUN --mount=type=cache natively
cat > Containerfile << 'EOF'
FROM docker.io/library/python:3.12-slim

WORKDIR /app
COPY requirements.txt .

# Use cache mount for pip downloads
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY . .
CMD ["python", "app.py"]
EOF

# Build with cache
podman build -t myapp .
# Subsequent builds will reuse the cached pip packages
```

## Docker Networking (Overlay Networks)

Docker overlay networks for multi-host communication are not available in Podman.

```bash
# Docker overlay network (requires Swarm)
# docker network create --driver overlay my-overlay

# Alternative 1: Use Podman with Kubernetes networking
# Deploy containers across hosts using Kubernetes

# Alternative 2: Use WireGuard or other VPN for multi-host networking
# Containers on different hosts connect via VPN tunnel

# Alternative 3: Use host networking with service discovery
podman run -d --network host --name myapp myimage

# For single-host multi-container networking, use Podman networks
podman network create mynet
podman run -d --network mynet --name web nginx
podman run -d --network mynet --name db postgres:16
```

## Docker Plugins (Volume and Network)

Docker supports third-party volume and network plugins. Podman does not have this plugin system.

```bash
# Docker volume plugins (not available in Podman)
# docker volume create --driver rexray/ebs myvolume

# Alternative: Use native mount options
# NFS volume
podman volume create \
  --opt type=nfs \
  --opt o=addr=nfs-server.example.com,rw \
  --opt device=:/exports/data \
  nfs-volume

# CIFS/SMB volume
podman volume create \
  --opt type=cifs \
  --opt o=username=user,password=pass \
  --opt device=//server/share \
  smb-volume

# tmpfs volume
podman volume create \
  --opt type=tmpfs \
  --opt o=size=100m \
  tmp-volume
```

## Docker Content Trust

Docker Content Trust (DCT) uses Notary for image signing. Podman uses a different approach.

```bash
# Docker Content Trust
# export DOCKER_CONTENT_TRUST=1
# docker push myimage:latest

# Podman alternative: Use Sigstore/cosign for image signing
# Install cosign
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Sign an image with cosign
cosign sign registry.example.com/myapp:latest

# Verify an image signature
cosign verify registry.example.com/myapp:latest

# Or use Skopeo with GPG signatures
skopeo copy \
  --sign-by "signer@example.com" \
  containers-storage:myapp:latest \
  docker://registry.example.com/myapp:latest
```

## Docker Events and Logging Drivers

Docker supports various logging drivers. Podman handles logging differently.

```bash
# Docker logging drivers (json-file, syslog, fluentd, etc.)
# docker run --log-driver=fluentd myimage

# Podman logging options
# Default: journald (on systemd systems) or k8s-file
podman run -d \
  --log-driver journald \
  --name myapp \
  myimage

# View logs through journalctl
journalctl CONTAINER_NAME=myapp

# Use k8s-file driver for file-based logging
podman run -d \
  --log-driver k8s-file \
  --log-opt path=/var/log/containers/myapp.log \
  --name myapp \
  myimage

# For fluentd-style log forwarding, use journald + journal-remote
# or run a log collector as a sidecar container
```

## Docker Configs and Secrets

Docker Swarm configs and secrets are not available in standalone Podman.

```bash
# Docker Swarm secrets (not available in Podman)
# docker secret create my_secret secret.txt
# docker service create --secret my_secret myimage

# Podman alternative: Use Podman secrets
podman secret create my_db_password - <<< "s3cretPassword"

# Use the secret in a container
podman run -d \
  --secret my_db_password \
  --name db \
  postgres:16

# The secret is mounted at /run/secrets/my_db_password
podman exec db cat /run/secrets/my_db_password

# List secrets
podman secret ls

# Remove a secret
podman secret rm my_db_password
```

## Summary

While Podman does not support every Docker-specific feature, it provides practical alternatives for each gap. Docker Swarm is replaced by Kubernetes integration and Podman pods. Restart policies are handled by systemd service generation. Overlay networks are addressed through Kubernetes or VPN solutions. Volume plugins are replaced by native mount options. Docker Content Trust is replaced by Sigstore/cosign or Skopeo GPG signing. Understanding these differences and their workarounds ensures a smooth transition from Docker to Podman without losing critical functionality.
