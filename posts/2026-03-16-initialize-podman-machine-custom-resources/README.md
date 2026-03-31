# How to Initialize a Podman Machine with Custom Resources

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Podman Machine, Virtual Machine, Container, DevOps

Description: Learn how to initialize a Podman machine with custom CPU, memory, and disk settings tailored to your development or production workloads.

---

> Customizing your Podman machine resources during initialization ensures your containers have the compute power they need from the start.

On macOS and Windows, Podman runs Linux containers inside a lightweight virtual machine called a Podman machine. By default, the machine is created with modest resources, but you can customize CPUs, memory, and disk size during initialization. This guide covers all the options available when setting up a Podman machine.

---

## Prerequisites

- Podman installed on macOS (via Homebrew) or Windows (via the installer)
- No existing Podman machine running (or willingness to recreate one)

## Understanding Podman Machines

Podman machines are lightweight Linux VMs that provide the Linux kernel needed to run containers on non-Linux systems:

```bash
# Check if any machines exist

podman machine list

# View default machine settings
podman machine info 2>/dev/null
```

## Step 1: Check Default Resource Allocation

Before customizing, see what the defaults are:

```bash
# Initialize a default machine (do not start it yet)
podman machine init --help
```

Default values typically are:
- CPUs: 1
- Memory: 2048 MB
- Disk Size: 100 GB

## Step 2: Initialize with Custom Resources

Create a machine with resources tailored to your needs:

```bash
# Initialize with 4 CPUs, 8GB RAM, and 100GB disk
podman machine init \
  --cpus 4 \
  --memory 8192 \
  --disk-size 100
```

For a development machine with more resources:

```bash
# Heavy development workload
podman machine init \
  --cpus 6 \
  --memory 16384 \
  --disk-size 200
```

For a lightweight machine:

```bash
# Minimal resources for simple container testing
podman machine init \
  --cpus 2 \
  --memory 2048 \
  --disk-size 30
```

## Step 3: Initialize Named Machines

Create multiple machines with different resource profiles:

```bash
# Create a lightweight machine for testing
podman machine init \
  --cpus 2 \
  --memory 2048 \
  --disk-size 30 \
  dev-light

# Create a powerful machine for building images
podman machine init \
  --cpus 8 \
  --memory 16384 \
  --disk-size 200 \
  dev-heavy

# List all machines
podman machine list
```

## Step 4: Initialize with a Custom VM Image

```bash
# Use a specific Fedora CoreOS image
podman machine init \
  --cpus 4 \
  --memory 8192 \
  --image /path/to/custom-image.qcow2

# Or specify an image URL
podman machine init \
  --cpus 4 \
  --memory 8192 \
  --image https://example.com/custom-vm.qcow2
```

## Step 5: Initialize with Volume Mounts

Pre-configure volume mounts during initialization:

```bash
# Mount your home directory into the machine
podman machine init \
  --cpus 4 \
  --memory 8192 \
  -v $HOME:$HOME

# Mount specific project directories
podman machine init \
  --cpus 4 \
  --memory 8192 \
  -v $HOME/projects:$HOME/projects
```

## Step 6: Initialize with Rootful Mode

By default, the Podman machine runs rootless. For rootful operation:

```bash
# Initialize with rootful mode enabled
podman machine init \
  --cpus 4 \
  --memory 8192 \
  --rootful

# This allows running containers as root inside the VM
```

## Step 7: Start the Machine

```bash
# Start the default machine
podman machine start

# Or start a named machine
podman machine start dev-heavy
```

## Step 8: Verify the Configuration

```bash
# Check machine details
podman machine inspect

# Verify resource allocation from inside the machine
podman machine ssh -- nproc           # Check CPU count
podman machine ssh -- free -h         # Check memory
podman machine ssh -- df -h /         # Check disk space
```

## Running a Practical Verification

Confirm the resources are working by running a resource-intensive task:

```bash
# Start the machine
podman machine start

# Run a container that uses multiple CPUs
podman run --rm docker.io/library/alpine:latest sh -c "
  echo 'CPUs available:' && nproc
  echo 'Memory available:' && free -h
  echo 'Disk available:' && df -h /
"

# Run a multi-process build to test CPU allocation
podman run --rm docker.io/library/golang:latest sh -c "
  echo 'Building with' \$(nproc) 'CPUs'
"
```

## Recreating a Machine with Different Resources

If you need to change resources after initialization:

```bash
# Stop the current machine
podman machine stop

# Remove it
podman machine rm

# Reinitialize with new resource settings
podman machine init \
  --cpus 8 \
  --memory 16384 \
  --disk-size 150

# Start the new machine
podman machine start

# Verify
podman machine inspect
```

Resource Planning Guidelines

Choose resources based on your workload:

```bash
# Light development (web apps, simple services)
# 2 CPUs, 4GB RAM, 50GB disk
podman machine init --cpus 2 --memory 4096 --disk-size 50

# Medium development (multi-container apps, databases)
# 4 CPUs, 8GB RAM, 100GB disk
podman machine init --cpus 4 --memory 8192 --disk-size 100

# Heavy development (image building, CI/CD, data processing)
# 8 CPUs, 16GB RAM, 200GB disk
podman machine init --cpus 8 --memory 16384 --disk-size 200
```

## Troubleshooting

If initialization fails with resource errors:

```bash
# Check your system's available resources
sysctl -n hw.ncpu            # macOS: total CPUs
sysctl -n hw.memsize          # macOS: total memory in bytes

# Do not allocate more than 75% of your system's resources
# For a system with 16GB RAM, use at most 12GB for the machine
```

If the machine starts but containers run out of memory:

```bash
# Check memory usage inside the machine
podman machine ssh -- free -h

# Recreate with more memory
podman machine stop
podman machine rm
podman machine init --cpus 4 --memory 12288 --disk-size 100
podman machine start
```

If disk space fills up:

```bash
# Check disk usage inside the machine
podman machine ssh -- df -h

# Prune unused resources
podman system prune -a

# If needed, recreate with larger disk
podman machine stop
podman machine rm
podman machine init --cpus 4 --memory 8192 --disk-size 200
podman machine start
```

## Summary

Initializing a Podman machine with custom resources ensures your containers have adequate compute power from the start. Use named machines to maintain multiple resource profiles for different workloads. When resource needs change, simply remove and reinitialize the machine with updated settings. Plan your resource allocation based on whether you are doing light development, running databases, or building container images.
