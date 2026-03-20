# How to Configure Default Environment Variables in containers.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Configuration, Environment Variables

Description: Learn how to set default environment variables in containers.conf so every Podman container inherits a consistent environment.

---

> Default environment variables in containers.conf ensure every container starts with a consistent baseline configuration without requiring explicit flags on each run.

Setting environment variables on every `podman run` command is tedious and error-prone. By defining defaults in `containers.conf`, you guarantee that all containers on your system start with the same baseline environment. This is particularly useful for proxy settings, locale configurations, and common paths. This guide shows you how to configure and manage default environment variables.

---

## Setting Basic Environment Variables

Define environment variables that apply to every container.

```bash
# Create or update user-level configuration

mkdir -p ~/.config/containers

cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
# Default environment variables applied to all containers
# These are injected into every container at creation time
env = [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "TERM=xterm-256color",
    "LANG=en_US.UTF-8",
    "LC_ALL=en_US.UTF-8"
]
EOF
```

```bash
# Verify the environment variables are applied
podman run --rm alpine env | sort
```

## Configuring Proxy Settings

Set corporate proxy variables that apply to all containers.

```bash
# Add proxy configuration to containers.conf
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
# Environment variables including proxy settings
env = [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "TERM=xterm-256color",
    "HTTP_PROXY=http://proxy.example.com:8080",
    "HTTPS_PROXY=http://proxy.example.com:8080",
    "NO_PROXY=localhost,127.0.0.1,.example.com",
    "http_proxy=http://proxy.example.com:8080",
    "https_proxy=http://proxy.example.com:8080",
    "no_proxy=localhost,127.0.0.1,.example.com"
]
EOF

# Verify proxy settings are injected
podman run --rm alpine env | grep -i proxy
```

## Using Host Environment Variables

Pass through specific host environment variables to containers.

```bash
# Configure environment variable passthrough
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
# Static default environment variables
env = [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "TERM=xterm-256color"
]

# Pass host environment variables into containers
# List variable names without values to pass the host value through
env_host = false
EOF

# If env_host is true, ALL host environment variables are passed
# This is generally not recommended for security reasons

# Instead, pass specific variables at runtime
podman run --rm -e USER -e HOME alpine env | grep -E "USER|HOME"
```

## Setting Application-Specific Defaults

Configure defaults for common application frameworks.

```bash
# Configure environment for Node.js development
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
env = [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "TERM=xterm-256color",
    "NODE_ENV=development",
    "NPM_CONFIG_LOGLEVEL=warn",
    "PYTHONDONTWRITEBYTECODE=1",
    "DEBIAN_FRONTEND=noninteractive"
]
EOF

# Test with a Node.js container
podman run --rm node:alpine env | grep NODE_ENV

# Test with a Python container
podman run --rm python:alpine env | grep PYTHONDONTWRITEBYTECODE
```

## Configuring Timezone Defaults

Set the timezone for all containers consistently.

```bash
# Use the tz option for timezone configuration
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
# Set timezone to match the host
# "local" uses the host timezone
# Or specify directly: "America/New_York", "Europe/London", etc.
tz = "local"

env = [
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "TERM=xterm-256color"
]
EOF

# Verify timezone in container matches host
echo "Host time: $(date)"
podman run --rm alpine date
```

## Overriding Default Variables Per Container

Override or extend defaults when running specific containers.

```bash
# Default env vars from containers.conf are always applied
# You can override them with -e flag
podman run --rm -e TERM=dumb alpine env | grep TERM

# Add additional variables beyond the defaults
podman run --rm -e MY_VAR=hello alpine env | grep MY_VAR

# Use --env-file for multiple overrides
cat > /tmp/my-env << 'EOF'
DATABASE_URL=postgres://localhost:5432/mydb
REDIS_URL=redis://localhost:6379
API_KEY=development-key
EOF

podman run --rm --env-file /tmp/my-env alpine env | grep -E "DATABASE|REDIS|API"

# Clean up
rm -f /tmp/my-env
```

## Debugging Environment Variable Issues

Troubleshoot when environment variables are not applied as expected.

```bash
# Check the effective containers.conf settings
podman --log-level=debug run --rm alpine env 2>&1 | head -20

# Compare default env with overridden env
echo "=== Default environment ==="
podman run --rm alpine env | sort

echo ""
echo "=== With override ==="
podman run --rm -e TERM=dumb -e CUSTOM=value alpine env | sort

# Verify which config file is being loaded
podman info --format '{{range .Host.ConfigFiles}}{{.}}{{"\n"}}{{end}}'
```

## Summary

Default environment variables in `containers.conf` provide a consistent baseline for every container created with Podman. Use them for proxy settings, locale configurations, timezone defaults, and framework-specific variables. The `env` array in the `[containers]` section accepts any environment variable, and runtime flags like `-e` and `--env-file` can override or extend these defaults on a per-container basis.
