# How to Configure Registry Search Order in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Registry, Search Order, Configuration

Description: Learn how to control the order in which Podman searches container registries when pulling images with unqualified names.

---

> The registry search order determines where Podman looks first when you pull an image without specifying a registry.

When you run `podman pull nginx` without a fully qualified registry name, Podman needs to know which registries to search and in what order. The `unqualified-search-registries` setting controls this behavior. Configuring the right search order ensures your team pulls from the correct sources and avoids accidentally pulling from unintended registries.

---

## Understanding Unqualified Image Names

An unqualified image name is one that does not include a registry prefix.

```bash
# Qualified name: includes the full registry path
podman pull docker.io/library/nginx:latest

# Unqualified name: no registry prefix
podman pull nginx:latest
# Podman must decide which registry to search
```

## Viewing the Current Search Order

Check which registries Podman currently searches.

```bash
# Display the current unqualified search registries
podman info --format '{{.Registries.Search}}'

# View the full registries section
podman info 2>&1 | grep -A 10 "registries"

# Read the raw configuration file
cat /etc/containers/registries.conf | grep unqualified
```

## Setting the Search Order

Edit the `registries.conf` file to define your preferred search order.

```bash
# Back up current configuration
sudo cp /etc/containers/registries.conf /etc/containers/registries.conf.bak

# Set the search order with your preferred registries
sudo tee /etc/containers/registries.conf <<'EOF'
# Registries are searched in order from left to right
# The first registry that has the image wins
unqualified-search-registries = ["docker.io", "quay.io", "ghcr.io"]
EOF
```

## Prioritizing an Internal Registry

Organizations often want their internal registry searched first.

```toml
# /etc/containers/registries.conf

# Internal registry is searched first, then public registries
unqualified-search-registries = [
  "internal-registry.company.com",
  "docker.io",
  "quay.io"
]
```

```bash
# Now pulling an unqualified image searches internal first
podman pull myapp:latest
# 1. Tries internal-registry.company.com/myapp:latest
# 2. Tries docker.io/library/myapp:latest
# 3. Tries quay.io/myapp:latest
```

## Single Registry Search

For strict environments, limit the search to one registry.

```toml
# /etc/containers/registries.conf

# Only search the approved internal registry
unqualified-search-registries = ["approved-registry.internal:5000"]
```

```bash
# All unqualified pulls go to the internal registry
podman pull alpine:latest
# Only checks approved-registry.internal:5000/alpine:latest
```

## Disabling Unqualified Search

You can disable unqualified search entirely, requiring fully qualified image names.

```toml
# /etc/containers/registries.conf

# Empty list disables unqualified search
unqualified-search-registries = []
```

```bash
# This will fail because no search registries are configured
podman pull nginx:latest
# Error: short-name resolution enforced

# This will work because it is fully qualified
podman pull docker.io/library/nginx:latest
```

## Short-Name Aliases

Podman also supports short-name aliases for common images, providing a way to map unqualified names to specific registries.

```bash
# View existing short-name aliases
cat /etc/containers/registries.conf.d/000-shortnames.conf | head -20

# The aliases file maps short names to fully qualified names
# For example: "nginx" = "docker.io/library/nginx"
```

```toml
# /etc/containers/registries.conf.d/custom-shortnames.conf

[aliases]
"myapp" = "internal-registry.company.com/team/myapp"
"nginx" = "docker.io/library/nginx"
"redis" = "docker.io/library/redis"
```

## User-Level Search Configuration

Individual users can override the system search order.

```bash
# Create user-level configuration
mkdir -p ~/.config/containers

cat > ~/.config/containers/registries.conf <<'EOF'
# User prefers ghcr.io over docker.io
unqualified-search-registries = ["ghcr.io", "docker.io", "quay.io"]
EOF

# Verify the user config takes effect
podman info --format '{{.Registries.Search}}'
```

## Debugging Search Order Issues

When images are not found or the wrong image is pulled, debug the search order.

```bash
# Use debug logging to see which registries are tried
podman --log-level=debug pull nginx:latest 2>&1 | grep -i "trying\|resolve\|search"

# Check if a specific image exists on a registry
skopeo inspect docker://docker.io/library/nginx:latest 2>/dev/null && echo "Found on docker.io"
skopeo inspect docker://quay.io/nginx:latest 2>/dev/null && echo "Found on quay.io"

# Verify the effective configuration
podman info --format '{{.Registries}}'
```

## Interactive Short-Name Resolution

When Podman encounters an ambiguous short name, it can prompt interactively.

```bash
# Podman may prompt when an image is found on multiple registries
podman pull nginx
# ? Please select an image:
#   docker.io/library/nginx:latest
#   quay.io/nginx:latest
# Select: 1

# To avoid the prompt, use the --short-name-alias-conf flag
# or configure aliases in the shortnames configuration
```

## Summary

The registry search order in Podman is controlled by the `unqualified-search-registries` list in `registries.conf`. Registries are tried in order from left to right when pulling images with unqualified names. You can prioritize internal registries, limit search to a single source, or disable unqualified search entirely to enforce fully qualified names. Short-name aliases provide additional control for mapping common image names to specific registries.
