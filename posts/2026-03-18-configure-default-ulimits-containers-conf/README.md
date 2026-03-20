# How to Configure Default Ulimits in containers.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Configuration, Resource Limits, Ulimit

Description: Learn how to configure default ulimits in containers.conf to set resource limits like file descriptors and process counts for all Podman containers.

---

> Default ulimits in containers.conf ensure every container starts with appropriate resource limits, preventing runaway processes from exhausting system resources.

Container resource limits (ulimits) control how many file descriptors, processes, and other system resources a container can use. Setting sensible defaults in `containers.conf` protects your host system while ensuring containers have enough resources to function. This guide covers configuring ulimits for various workloads.

---

## Understanding Ulimits

Ulimits define soft and hard resource limits for containers.

```bash
# View current default ulimits in Podman

podman run --rm alpine sh -c 'ulimit -a'

# Key ulimit types:
# nofile  - Maximum open file descriptors
# nproc   - Maximum number of processes
# memlock - Maximum locked memory (bytes)
# core    - Maximum core file size
# stack   - Maximum stack size
# as      - Maximum address space
```

## Setting Default Ulimits

Configure default ulimits in containers.conf.

```bash
# Create or update user-level configuration
mkdir -p ~/.config/containers

cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
# Default ulimits for all containers
# Format: "type=soft:hard"
# soft = default limit, hard = maximum limit
default_ulimits = [
    "nofile=65536:65536",
    "nproc=4096:8192"
]
EOF

# Verify the ulimits are applied
podman run --rm alpine sh -c 'ulimit -n && ulimit -u'
```

## Configuring File Descriptor Limits

Set appropriate limits for applications that open many files.

```bash
# Configure high file descriptor limits for database workloads
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
default_ulimits = [
    # High file descriptor limit for databases and web servers
    # Soft limit: 65536, Hard limit: 65536
    "nofile=65536:65536",
    "nproc=4096:8192"
]
EOF

# Test with a container that needs many file descriptors
podman run --rm alpine sh -c '
    echo "Open files soft limit: $(ulimit -Sn)"
    echo "Open files hard limit: $(ulimit -Hn)"
'

# Compare with default (usually 1024)
# Without the setting, this would show much lower values
```

## Configuring Process Limits

Control the maximum number of processes per container.

```bash
# Set process limits to prevent fork bombs
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
default_ulimits = [
    "nofile=65536:65536",
    # Limit processes to prevent fork bombs
    # Soft: 4096, Hard: 8192
    "nproc=4096:8192",
    # Limit core dump size (0 = disabled)
    "core=0:0"
]
EOF

# Verify process limits
podman run --rm alpine sh -c '
    echo "Max processes soft: $(ulimit -Su)"
    echo "Max processes hard: $(ulimit -Hu)"
    echo "Core dump size: $(ulimit -c)"
'
```

## Configuring Memory Limits

Set memory-related ulimits for memory-intensive workloads.

```bash
# Configure memory-related ulimits
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
default_ulimits = [
    "nofile=65536:65536",
    "nproc=4096:8192",
    # Maximum locked memory in bytes (64MB)
    "memlock=67108864:67108864",
    # Maximum stack size in bytes (8MB)
    "stack=8388608:8388608"
]
EOF

# Verify memory limits
podman run --rm alpine sh -c '
    echo "Locked memory: $(ulimit -l) KB"
    echo "Stack size: $(ulimit -s) KB"
'
```

## Workload-Specific Ulimit Profiles

Different workloads require different ulimit configurations.

```bash
# Web server / API workload profile
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
# Balanced profile for web applications
default_ulimits = [
    "nofile=65536:65536",
    "nproc=4096:8192",
    "core=0:0"
]
EOF

# For database workloads, you might want even higher limits
# Override per container at runtime:
podman run --rm --ulimit nofile=131072:131072 alpine sh -c 'ulimit -n'

# For batch processing jobs with strict limits:
podman run --rm --ulimit nproc=256:512 --ulimit nofile=1024:2048 alpine sh -c '
    echo "Files: $(ulimit -n)"
    echo "Procs: $(ulimit -u)"
'
```

## Overriding Ulimits at Runtime

Override default ulimits for specific containers.

```bash
# Set conservative defaults in config
cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
default_ulimits = [
    "nofile=1024:4096",
    "nproc=512:1024"
]
EOF

# Override for a database container that needs more resources
podman run --rm \
    --ulimit nofile=65536:65536 \
    --ulimit nproc=4096:4096 \
    alpine sh -c '
        echo "File descriptors: $(ulimit -n)"
        echo "Processes: $(ulimit -u)"
    '

# Override for a minimal container with strict limits
podman run --rm \
    --ulimit nofile=256:256 \
    --ulimit nproc=64:64 \
    alpine sh -c '
        echo "File descriptors: $(ulimit -n)"
        echo "Processes: $(ulimit -u)"
    '
```

## Troubleshooting Ulimit Issues

Debug common problems with ulimit configurations.

```bash
# Check if ulimits are being applied from config
podman run --rm alpine sh -c 'ulimit -a' 2>&1

# Verify the host system limits (container cannot exceed these)
ulimit -a

# If container ulimits are lower than expected, check host limits
echo "Host nofile limit: $(ulimit -n)"
echo "Host nproc limit: $(ulimit -u)"

# Debug with verbose Podman output
podman --log-level=debug run --rm alpine ulimit -n 2>&1 | grep -i ulimit | head -5

# Verify configuration syntax
podman info > /dev/null 2>&1 && echo "Config valid" || echo "Config error"
```

## Summary

Default ulimits in `containers.conf` set resource boundaries for all containers, protecting the host from resource exhaustion while ensuring containers have enough resources. Configure `nofile` for file-heavy workloads, `nproc` to prevent fork bombs, and `memlock` for memory-intensive applications. Use the `--ulimit` flag to override defaults for specific containers. Always ensure container ulimits do not exceed host system limits.
