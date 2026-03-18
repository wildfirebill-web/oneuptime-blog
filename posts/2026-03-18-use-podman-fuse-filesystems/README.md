# How to Use Podman with FUSE Filesystems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, FUSE, Filesystem, Storage, Linux

Description: A practical guide to using FUSE filesystems with Podman containers, including fuse-overlayfs for rootless containers, SSHFS mounts, and custom FUSE filesystem integration.

---

> FUSE (Filesystem in Userspace) enables Podman's rootless container support and opens up powerful storage options. Understanding how Podman uses FUSE filesystems helps you optimize performance and unlock advanced storage patterns.

FUSE allows filesystems to be implemented in userspace rather than kernel space. Podman relies heavily on FUSE for its rootless mode through fuse-overlayfs, and you can mount additional FUSE filesystems inside containers for scenarios like remote storage access, encrypted volumes, and custom file systems.

---

## Understanding FUSE in Podman

Podman uses FUSE in two primary ways:

1. **fuse-overlayfs**: The overlay filesystem driver for rootless containers
2. **Container FUSE mounts**: Mounting FUSE filesystems inside containers

### Why FUSE Matters for Rootless Podman

The standard overlay filesystem requires kernel privileges. Rootless Podman cannot use the kernel overlay driver, so it relies on fuse-overlayfs to provide the same copy-on-write semantics in userspace.

Check your current storage driver:

```bash
podman info --format '{{.Store.GraphDriverName}}'
```

If you are running rootless and see "overlay," it is likely using fuse-overlayfs underneath:

```bash
podman info --format '{{.Store.GraphOptions}}'
```

## Installing fuse-overlayfs

On most systems, fuse-overlayfs is installed alongside Podman. If not, install it manually:

```bash
# Fedora/RHEL
sudo dnf install fuse-overlayfs

# Ubuntu/Debian
sudo apt-get install fuse-overlayfs

# Verify installation
fuse-overlayfs --version
```

## Configuring Podman Storage with FUSE

### Storage Configuration File

Configure Podman to use fuse-overlayfs in the storage configuration:

```bash
mkdir -p ~/.config/containers
```

```ini
# ~/.config/containers/storage.conf
[storage]
driver = "overlay"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
mountopt = "nodev,metacopy=on"
```

Verify the configuration:

```bash
podman system reset  # Warning: removes all containers and images
podman info --format '{{.Store.GraphDriverName}}'
```

### Performance Tuning

Optimize fuse-overlayfs performance with mount options:

```ini
# ~/.config/containers/storage.conf
[storage]
driver = "overlay"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
# Enable metacopy for faster file operations
mountopt = "nodev,metacopy=on"
```

## Mounting FUSE Filesystems Inside Containers

To use FUSE filesystems inside containers, you need access to the `/dev/fuse` device:

```bash
podman run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  alpine sh
```

### Using SSHFS Inside a Container

Mount a remote filesystem via SSH inside a container:

```dockerfile
# Containerfile.sshfs
FROM fedora:latest

RUN dnf install -y sshfs openssh-clients && dnf clean all

RUN mkdir /remote
ENTRYPOINT ["/bin/bash"]
```

Build and run:

```bash
podman build -t sshfs-container -f Containerfile.sshfs .

podman run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  -v ~/.ssh/id_rsa:/root/.ssh/id_rsa:ro \
  sshfs-container \
  -c '
    sshfs user@remote-host:/data /remote -o StrictHostKeyChecking=no
    ls /remote
    fusermount -u /remote
  '
```

### Using S3 FUSE Inside a Container

Mount an S3 bucket as a filesystem:

```dockerfile
# Containerfile.s3fs
FROM fedora:latest

RUN dnf install -y s3fs-fuse && dnf clean all

RUN mkdir /s3data
ENTRYPOINT ["/bin/bash"]
```

```bash
podman build -t s3fs-container -f Containerfile.s3fs .

podman run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  -e AWS_ACCESS_KEY_ID=your-key \
  -e AWS_SECRET_ACCESS_KEY=your-secret \
  s3fs-container \
  -c '
    echo "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" > /etc/passwd-s3fs
    chmod 600 /etc/passwd-s3fs
    s3fs my-bucket /s3data -o passwd_file=/etc/passwd-s3fs -o url=https://s3.amazonaws.com
    ls /s3data
    fusermount -u /s3data
  '
```

## GlusterFS with Podman

Mount a GlusterFS volume inside a container:

```bash
podman run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --network host \
  fedora:latest \
  bash -c '
    dnf install -y glusterfs-fuse
    mkdir /gluster
    mount -t glusterfs gluster-server:/volume /gluster
    ls /gluster
  '
```

## Encrypted FUSE Filesystems

Use gocryptfs or encfs for encrypted storage inside containers:

```dockerfile
# Containerfile.encrypted
FROM fedora:latest

RUN dnf install -y gocryptfs && dnf clean all

RUN mkdir -p /encrypted /decrypted
ENTRYPOINT ["/bin/bash"]
```

```bash
podman build -t encrypted-container -f Containerfile.encrypted .

# Initialize encrypted filesystem on host
mkdir -p /tmp/encrypted-data
echo "mypassword" | gocryptfs -init -passfile /dev/stdin /tmp/encrypted-data

# Run container with encrypted mount
podman run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  -v /tmp/encrypted-data:/encrypted:Z \
  encrypted-container \
  -c '
    echo "mypassword" | gocryptfs -passfile /dev/stdin /encrypted /decrypted
    echo "Secret data" > /decrypted/secret.txt
    cat /decrypted/secret.txt
    fusermount -u /decrypted
  '
```

## Troubleshooting FUSE with Podman

### /dev/fuse Not Available

If the FUSE device is not present:

```bash
# Check if FUSE is loaded
lsmod | grep fuse

# Load the FUSE module
sudo modprobe fuse

# Verify /dev/fuse exists
ls -la /dev/fuse
```

### Permission Denied on FUSE Mount

For rootless containers, ensure proper capabilities:

```bash
podman run --rm -it \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor=unconfined \
  alpine sh -c 'ls -la /dev/fuse'
```

### fuse-overlayfs Performance Issues

If container operations are slow, check fuse-overlayfs performance:

```bash
# Benchmark layer operations
time podman pull docker.io/library/python:3.11

# Check for excessive I/O
podman system df

# Consider using native overlay if running as root
sudo podman info --format '{{.Store.GraphDriverName}}'
```

### Storage Driver Fallback

If fuse-overlayfs is not available, Podman falls back to the VFS driver. Check and correct this:

```bash
# Check current driver
podman info --format '{{.Store.GraphDriverName}}'

# If VFS, install fuse-overlayfs
sudo dnf install fuse-overlayfs

# Reset storage to apply new driver
podman system reset
```

## Comparing Storage Drivers

Here is a practical comparison script:

```python
#!/usr/bin/env python3
"""Compare Podman storage driver performance."""

import subprocess
import time

def benchmark_operation(description, command):
    """Time a podman operation."""
    start = time.time()
    result = subprocess.run(command, shell=True, capture_output=True, text=True)
    elapsed = time.time() - start

    status = "OK" if result.returncode == 0 else "FAIL"
    print(f"  {description}: {elapsed:.2f}s [{status}]")
    return elapsed

def run_benchmarks():
    """Run storage performance benchmarks."""
    print("Storage Driver Benchmarks")
    print("=" * 50)

    # Get current driver
    result = subprocess.run(
        ["podman", "info", "--format", "{{.Store.GraphDriverName}}"],
        capture_output=True, text=True
    )
    driver = result.stdout.strip()
    print(f"Driver: {driver}\n")

    # Pull benchmark
    subprocess.run(["podman", "rmi", "-f", "alpine:latest"],
                   capture_output=True)
    benchmark_operation("Pull alpine:latest",
                       "podman pull docker.io/library/alpine:latest")

    # Create container benchmark
    benchmark_operation("Create container",
                       "podman create --name bench-test alpine:latest echo test")

    # Start container benchmark
    benchmark_operation("Start container",
                       "podman start bench-test")

    # Remove container
    benchmark_operation("Remove container",
                       "podman rm -f bench-test")

    # Build benchmark
    benchmark_operation("Build simple image",
                       'echo "FROM alpine\nRUN echo hello" | podman build -t bench-image -f - .')

    # Cleanup
    subprocess.run(["podman", "rmi", "-f", "bench-image"], capture_output=True)

run_benchmarks()
```

## Best Practices

### Security Recommendations

When using FUSE inside containers, follow these security guidelines:

```bash
# Prefer --device /dev/fuse over --privileged
podman run --device /dev/fuse --cap-add SYS_ADMIN ...

# Use read-only root filesystem where possible
podman run --device /dev/fuse --cap-add SYS_ADMIN \
  --read-only --tmpfs /tmp \
  my-fuse-container

# Drop unnecessary capabilities
podman run --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --cap-drop ALL \
  --cap-add SYS_ADMIN \
  my-fuse-container
```

### Performance Tips

```bash
# Use native overlay when running as root
sudo podman run ...  # Uses kernel overlay, fastest option

# For rootless, ensure fuse-overlayfs is current
fuse-overlayfs --version  # Check for latest version

# Use metacopy option for faster operations
# In storage.conf: mountopt = "nodev,metacopy=on"

# Use tmpfs for ephemeral containers
podman run --tmpfs /var/lib/data:rw,size=2g ...
```

## Conclusion

FUSE filesystems are fundamental to Podman's rootless container support through fuse-overlayfs, and they enable powerful storage patterns when mounted inside containers. From remote filesystem access with SSHFS to encrypted storage with gocryptfs, FUSE expands what is possible with containerized applications. Understanding how to configure, optimize, and troubleshoot FUSE with Podman ensures you get the best performance and functionality from your container workloads.
