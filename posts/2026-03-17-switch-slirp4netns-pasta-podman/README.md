# How to Switch from slirp4netns to Pasta in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Networking, Pasta, slirp4netns, Migration

Description: Learn how to migrate from slirp4netns to pasta networking in Podman for better performance and features.

---

> Switching from slirp4netns to pasta improves rootless container networking performance, adds native IPv6 support, and reduces memory overhead.

Pasta is the recommended replacement for slirp4netns in rootless Podman. The migration is straightforward and brings significant networking improvements. This guide covers the steps to switch and verify the new backend.

---

## Checking Your Current Backend

```bash
# Check which backend is currently in use
podman info --format '{{ .Host.NetworkBackend }}'

# Check available backends
podman info --format 'Pasta: {{ .Host.Pasta.Executable }}'
podman info --format 'slirp4netns: {{ .Host.Slirp4NetNs.Executable }}'
```

## Installing Pasta

```bash
# On Fedora/RHEL
sudo dnf install passt -y

# On Ubuntu/Debian
sudo apt install passt -y

# On Arch Linux
sudo pacman -S passt

# Verify installation
pasta --version
```

## Switching the Default Backend

```bash
# Create or edit the containers.conf file
mkdir -p ~/.config/containers

cat >> ~/.config/containers/containers.conf << 'EOF'
[containers]
default_rootless_network_cmd = "pasta"
EOF

# Verify the change
podman info --format '{{ .Host.NetworkBackend }}'
```

## Testing Pasta with Existing Containers

```bash
# Stop running containers
podman stop --all

# Restart a container (it will use the new backend)
podman start web

# Or run a new test container
podman run --rm --network pasta \
  docker.io/library/alpine:latest sh -c "
    echo '=== Network Config ==='
    ip addr show
    echo '=== Connectivity ==='
    ping -c 2 8.8.8.8
    echo '=== DNS ==='
    nslookup google.com
  "
```

## Comparing Performance

```bash
# Test with slirp4netns
podman run --rm --network slirp4netns \
  docker.io/library/alpine:latest sh -c "
    apk add --no-cache curl > /dev/null 2>&1
    time curl -s -o /dev/null http://speedtest.tele2.net/1MB.zip
  "

# Test with pasta
podman run --rm --network pasta \
  docker.io/library/alpine:latest sh -c "
    apk add --no-cache curl > /dev/null 2>&1
    time curl -s -o /dev/null http://speedtest.tele2.net/1MB.zip
  "
```

## Verifying Port Forwarding Works

```bash
# Test port forwarding with pasta
podman run -d --name test-web \
  --network pasta \
  -p 8080:80 \
  docker.io/library/nginx:latest

# Verify the port is published
podman port test-web
curl -s http://localhost:8080

# Clean up
podman rm -f test-web
```

## Handling Migration Issues

```bash
# If containers fail to start after switching, recreate them
podman rm -f mycontainer
podman run -d --name mycontainer \
  --network pasta \
  -p 8080:80 \
  docker.io/library/nginx:latest

# If pasta is not working, temporarily fall back to slirp4netns
podman run -d --name fallback \
  --network slirp4netns \
  -p 8081:80 \
  docker.io/library/nginx:latest
```

## Reverting to slirp4netns

```bash
# If you need to revert, update containers.conf
# Change the line to:
# default_rootless_network_cmd = "slirp4netns"

# Or remove the setting to use Podman's default
sed -i '/default_rootless_network_cmd/d' ~/.config/containers/containers.conf
```

## Key Improvements After Switching

| Feature | slirp4netns | Pasta |
|---------|-------------|-------|
| Throughput | ~1 Gbps | ~5+ Gbps |
| IPv6 | Limited | Native |
| Memory per container | Higher | Lower |
| Startup time | Slower | Faster |

## Summary

Switch from slirp4netns to pasta by installing the `passt` package and setting `default_rootless_network_cmd = "pasta"` in `~/.config/containers/containers.conf`. Pasta provides better throughput, native IPv6 support, lower memory usage, and faster startup. Test port forwarding and connectivity after switching, and fall back to slirp4netns per-container if any issues arise during migration.
