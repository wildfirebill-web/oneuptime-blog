# How to Block a Registry in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Registry, Security, Policy

Description: Learn how to block specific container registries in Podman to prevent image pulls from untrusted or unauthorized sources.

---

> Blocking registries is a key security measure to ensure containers only come from approved sources.

In enterprise and security-sensitive environments, controlling which registries users can pull images from is critical. Podman allows administrators to block specific registries entirely, preventing any image pulls or pushes to those locations. This guide covers how to block registries, verify the blocks are active, and combine blocking with other security policies.

---

## Why Block Registries

Blocking registries serves several important purposes.

```bash
# Reasons to block registries:
# - Prevent pulling untrusted or unvetted images
# - Enforce use of approved internal registries only
# - Comply with organizational security policies
# - Reduce attack surface by limiting image sources
# - Prevent accidental use of public registries in production
```

## Blocking a Registry in registries.conf

The `blocked = true` setting prevents Podman from pulling images from a registry.

```bash
# Back up the configuration
sudo cp /etc/containers/registries.conf /etc/containers/registries.conf.bak

# Add a blocked registry
sudo tee -a /etc/containers/registries.conf <<'EOF'

# Block untrusted registries
[[registry]]
prefix = "untrusted-registry.example.com"
blocked = true
EOF
```

## Blocking Multiple Registries

You can block several registries by adding multiple blocks.

```toml
# /etc/containers/registries.conf

unqualified-search-registries = ["approved-registry.internal"]

# Block Docker Hub
[[registry]]
prefix = "docker.io"
blocked = true

# Block Quay.io
[[registry]]
prefix = "quay.io"
blocked = true

# Block GitHub Container Registry
[[registry]]
prefix = "ghcr.io"
blocked = true

# Only allow the internal registry
[[registry]]
prefix = "approved-registry.internal"
location = "approved-registry.internal"
insecure = false
```

## Testing Registry Blocks

Verify that blocked registries cannot be used.

```bash
# Attempt to pull from a blocked registry
podman pull docker.io/library/alpine:latest
# Expected error: "registry docker.io is blocked"

# Verify the error message
podman pull docker.io/library/nginx:latest 2>&1 | grep -i "blocked"

# Confirm that approved registries still work
podman pull approved-registry.internal/alpine:latest
```

## Blocking with Pattern Matching

You can block registries using prefix patterns to cover entire domains.

```toml
# /etc/containers/registries.conf

# Block all registries under a specific domain
[[registry]]
prefix = "*.untrusted-domain.com"
blocked = true

# Block a registry and all its subpaths
[[registry]]
prefix = "public-registry.io"
blocked = true
```

## Combining Blocks with an Allow List

A common pattern is to block all public registries and only allow specific internal ones.

```toml
# /etc/containers/registries.conf

# Only search the approved registry
unqualified-search-registries = ["approved.internal:5000"]

# Allow the approved internal registry
[[registry]]
prefix = "approved.internal:5000"
location = "approved.internal:5000"
insecure = false

# Block everything else
[[registry]]
prefix = "docker.io"
blocked = true

[[registry]]
prefix = "quay.io"
blocked = true

[[registry]]
prefix = "ghcr.io"
blocked = true

[[registry]]
prefix = "gcr.io"
blocked = true

[[registry]]
prefix = "mcr.microsoft.com"
blocked = true
```

## System-Wide Enforcement

Ensure the block applies to all users on the system.

```bash
# Set strict permissions on the system configuration
sudo chmod 644 /etc/containers/registries.conf
sudo chown root:root /etc/containers/registries.conf

# Prevent users from overriding with user-level config
# by setting a restrictive policy (optional, depends on policy)
# Note: user-level configs take precedence by default
```

## Verifying Block Configuration

Check the effective registry configuration.

```bash
# Display all registry settings
podman info --format '{{.Registries}}'

# Check with debug logging for detailed output
podman --log-level=debug pull docker.io/library/alpine:latest 2>&1 | head -20

# List all configured registries and their status
podman info 2>&1 | grep -A 5 -i "blocked"
```

## Temporarily Unblocking a Registry

If you need to temporarily unblock a registry for maintenance.

```bash
# Create a temporary configuration that unblocks the registry
sudo cp /etc/containers/registries.conf /etc/containers/registries.conf.blocked

# Edit to remove or comment out the blocked line
sudo sed -i 's/blocked = true/blocked = false/' /etc/containers/registries.conf

# Perform the needed pull
podman pull docker.io/library/alpine:latest

# Restore the blocked configuration
sudo cp /etc/containers/registries.conf.blocked /etc/containers/registries.conf
```

## Summary

Blocking registries in Podman is done by setting `blocked = true` in the `registries.conf` file for each registry you want to restrict. This is a fundamental security measure for environments that need to control where container images come from. Combine blocking with an approved registry allow list for the strongest security posture. Always test your blocks after configuration and ensure system-level permissions prevent unauthorized overrides.
