# How to Use the --userns=auto Option in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Rootless, User Namespaces, Auto Mapping

Description: Learn how to use the --userns=auto option in Podman to automatically allocate unique UID ranges for maximum container isolation.

---

> The `--userns=auto` option automatically assigns each container a unique UID range from your subordinate IDs, ensuring containers cannot access each other's files even if compromised.

By default, all rootless containers share the same UID mapping. If one container is compromised, it could potentially access files created by another container. The `auto` mode solves this by giving each container its own unique, non-overlapping UID range.

---

## Basic Auto Mode Usage

```bash
# Run a container with automatic UID allocation

podman run --rm --userns=auto alpine:latest cat /proc/self/uid_map

# Each new container gets a different range
podman run --rm --userns=auto alpine:latest cat /proc/self/uid_map
# Notice the host ID is different from the first container
```

## Demonstrating Container Isolation

```bash
# Start two containers with auto mode
podman run -d --name auto1 --userns=auto alpine:latest sleep 300
podman run -d --name auto2 --userns=auto alpine:latest sleep 300

# View their UID maps - they should be different
podman exec auto1 cat /proc/self/uid_map
podman exec auto2 cat /proc/self/uid_map

# Files created by one container cannot be read by the other
# because they map to different host UIDs

podman rm -f auto1 auto2
```

## Configuring the Auto Range Size

```bash
# By default, auto allocates 65536 UIDs per container
# You can specify a smaller range to fit more containers
podman run --rm --userns=auto:size=10000 alpine:latest cat /proc/self/uid_map

# Smaller ranges allow more concurrent containers
# but some images may need more UIDs

# Specify a larger range for images with many users
podman run --rm --userns=auto:size=131072 alpine:latest cat /proc/self/uid_map
```

## Prerequisites for Auto Mode

```bash
# Auto mode requires a large subuid/subgid range
# Check your current allocation
grep "$USER" /etc/subuid

# For running multiple containers with auto, you need enough IDs
# Example: 10 containers * 65536 UIDs = 655360 UIDs needed
# Ensure your subuid range is large enough
sudo usermod --del-subuids 100000-165535 $USER
sudo usermod --add-subuids 100000-1065535 $USER
sudo usermod --del-subgids 100000-165535 $USER
sudo usermod --add-subgids 100000-1065535 $USER

# Apply changes
podman system migrate
```

## Auto Mode with Services

```bash
# Run isolated service containers
podman run -d \
  --name web-isolated \
  --userns=auto \
  -p 8080:8080 \
  my-web-app:latest

podman run -d \
  --name db-isolated \
  --userns=auto \
  -v dbdata:/var/lib/postgresql/data \
  postgres:15

# Each service has its own UID range
# A compromise of the web container cannot access database files
```

## Configuring Auto Mode Globally

```bash
# Set auto mode as the default in containers.conf
mkdir -p ~/.config/containers

cat >> ~/.config/containers/containers.conf << 'EOF'
[containers]
userns = "auto"
EOF

# Now all containers use auto mode by default
podman run --rm alpine:latest cat /proc/self/uid_map
```

## When to Use Auto Mode

```bash
# Multi-tenant environments: isolate containers from different users/projects
podman run -d --userns=auto --name tenant-a-app tenant-a:latest
podman run -d --userns=auto --name tenant-b-app tenant-b:latest

# Security-sensitive workloads: prevent cross-container access
podman run -d --userns=auto --name payment-service payment:latest
podman run -d --userns=auto --name frontend frontend:latest

# CI/CD build isolation: prevent build jobs from interfering
podman run --rm --userns=auto builder:latest make test
podman run --rm --userns=auto builder:latest make integration-test
```

## Comparing Auto with Other Modes

```bash
# Default (no --userns): all containers share the same UID mapping
# Less isolation, but simpler for volume sharing between containers

# keep-id: maps your host UID into the container
# Great for development, not for inter-container isolation

# auto: unique UID range per container
# Maximum isolation, ideal for production and multi-tenant
```

## Summary

The `--userns=auto` option provides maximum inter-container isolation by assigning each container a unique, non-overlapping UID range from your subordinate ID allocation. This prevents a compromised container from accessing files belonging to other containers. It requires a sufficiently large subuid/subgid range and can be configured globally in `containers.conf`. Use auto mode for production workloads, multi-tenant environments, and any scenario where container isolation is a priority.
