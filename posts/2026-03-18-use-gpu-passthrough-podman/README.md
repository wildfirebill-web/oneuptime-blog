# How to Use GPU Passthrough with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, GPU, Containers, DevOps, Linux

Description: Learn how to pass through GPUs to Podman containers for hardware-accelerated workloads including machine learning, video processing, and scientific computing.

---

> GPU passthrough in Podman unlocks the full power of your hardware inside containers, enabling accelerated computing without the overhead of virtualization.

Running GPU-accelerated workloads inside containers has become essential for modern development workflows. Whether you are training machine learning models, rendering video, or running scientific simulations, having direct access to the GPU from within a container can dramatically improve performance. Podman, a daemonless container engine, provides several mechanisms to pass GPU devices into containers. This guide covers the core concepts, setup steps, and practical examples for GPU passthrough with Podman.

---

## Understanding GPU Passthrough

GPU passthrough allows a container to directly access the host's GPU hardware. Unlike CPU-based emulation, passthrough gives the container near-native performance by mapping the GPU device files and driver stack into the container's namespace.

There are two primary approaches to GPU passthrough in Podman:

1. **Device file mapping** - Passing `/dev/dri/*` or `/dev/nvidia*` device files directly into the container.
2. **CDI (Container Device Interface)** - A standardized specification for describing how devices should be made available to containers.

Before diving in, ensure your host system has the appropriate GPU drivers installed.

## Prerequisites

You need a Linux host with the GPU drivers properly installed. Verify your GPU is recognized by the system:

```bash
# Check for available GPU devices
ls -la /dev/dri/
# Example output:
# drwxr-xr-x  3 root root       120 Mar 18 08:00 .
# crw-rw----+ 1 root video  226,   0 Mar 18 08:00 card0
# crw-rw----+ 1 root render 226, 128 Mar 18 08:00 renderD128

# For NVIDIA GPUs, also check:
ls -la /dev/nvidia*
# Example output:
# crw-rw-rw- 1 root root 195,   0 Mar 18 08:00 /dev/nvidia0
# crw-rw-rw- 1 root root 195, 255 Mar 18 08:00 /dev/nvidiactl
# crw-rw-rw- 1 root root 195, 254 Mar 18 08:00 /dev/nvidia-modeset
# crw-rw-rw- 1 root root 509,   0 Mar 18 08:00 /dev/nvidia-uvm
```

Install Podman if it is not already present:

```bash
# On Fedora/RHEL/CentOS
sudo dnf install -y podman

# On Ubuntu/Debian
sudo apt-get install -y podman

# Verify installation
podman --version
```

## Passing GPU Devices with the --device Flag

The most straightforward method is using the `--device` flag to map GPU device files into the container.

### Intel and AMD GPUs (DRI Devices)

For Intel and AMD GPUs that use the Direct Rendering Infrastructure:

```bash
# Pass the entire /dev/dri directory to the container
podman run --rm -it \
  --device /dev/dri:/dev/dri \
  fedora:latest \
  bash -c "ls -la /dev/dri/"
```

You can also pass specific device nodes if you only need render access:

```bash
# Pass only the render node for compute workloads
podman run --rm -it \
  --device /dev/dri/renderD128:/dev/dri/renderD128 \
  my-compute-image:latest \
  python3 run_inference.py
```

### NVIDIA GPUs

For NVIDIA GPUs, you need to pass multiple device files:

```bash
# Pass all NVIDIA device files
podman run --rm -it \
  --device /dev/nvidia0:/dev/nvidia0 \
  --device /dev/nvidiactl:/dev/nvidiactl \
  --device /dev/nvidia-modeset:/dev/nvidia-modeset \
  --device /dev/nvidia-uvm:/dev/nvidia-uvm \
  --device /dev/nvidia-uvm-tools:/dev/nvidia-uvm-tools \
  nvidia/cuda:12.3.0-runtime-ubuntu22.04 \
  nvidia-smi
```

### Multi-GPU Systems

If your system has multiple GPUs, you can selectively pass specific GPUs:

```bash
# Pass only the first GPU (index 0)
podman run --rm -it \
  --device /dev/nvidia0:/dev/nvidia0 \
  --device /dev/nvidiactl:/dev/nvidiactl \
  --device /dev/nvidia-uvm:/dev/nvidia-uvm \
  nvidia/cuda:12.3.0-runtime-ubuntu22.04 \
  nvidia-smi

# Pass both GPUs on a dual-GPU system
podman run --rm -it \
  --device /dev/nvidia0:/dev/nvidia0 \
  --device /dev/nvidia1:/dev/nvidia1 \
  --device /dev/nvidiactl:/dev/nvidiactl \
  --device /dev/nvidia-uvm:/dev/nvidia-uvm \
  nvidia/cuda:12.3.0-runtime-ubuntu22.04 \
  nvidia-smi
```

## Using the NVIDIA Container Toolkit with Podman

The NVIDIA Container Toolkit provides a more integrated experience by automatically handling device mapping and driver library mounting.

```bash
# Install the NVIDIA Container Toolkit
# Add the repository
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo

# Install the toolkit
sudo dnf install -y nvidia-container-toolkit

# Generate CDI specification for your GPUs
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Verify the CDI spec was created
nvidia-ctk cdi list
```

Once the CDI spec is generated, you can use the `--device` flag with CDI identifiers:

```bash
# Run a container with all NVIDIA GPUs using CDI
podman run --rm -it \
  --device nvidia.com/gpu=all \
  nvidia/cuda:12.3.0-runtime-ubuntu22.04 \
  nvidia-smi

# Run with a specific GPU using CDI
podman run --rm -it \
  --device nvidia.com/gpu=0 \
  nvidia/cuda:12.3.0-runtime-ubuntu22.04 \
  nvidia-smi
```

## Security Considerations

When passing GPUs through to containers, keep these security practices in mind:

```bash
# Run rootless containers when possible for better isolation
podman run --rm -it \
  --device /dev/dri:/dev/dri \
  --user 1000:1000 \
  my-gpu-image:latest \
  python3 my_script.py

# Use --security-opt to apply SELinux labels if needed
podman run --rm -it \
  --device /dev/dri:/dev/dri \
  --security-opt label=type:container_runtime_t \
  my-gpu-image:latest \
  python3 my_script.py
```

For rootless Podman, the user running the container must have permission to access the GPU device files. Add the user to the appropriate group:

```bash
# For Intel/AMD GPUs (DRI devices)
sudo usermod -aG video $USER
sudo usermod -aG render $USER

# Log out and back in for group changes to take effect
```

## Verifying GPU Access Inside the Container

Once your container is running, verify that the GPU is accessible:

```bash
# For NVIDIA GPUs - run nvidia-smi
podman run --rm -it \
  --device nvidia.com/gpu=all \
  nvidia/cuda:12.3.0-runtime-ubuntu22.04 \
  nvidia-smi

# For Intel GPUs - check with vainfo
podman run --rm -it \
  --device /dev/dri:/dev/dri \
  intel-media-va-driver:latest \
  vainfo

# For AMD GPUs - check with clinfo
podman run --rm -it \
  --device /dev/dri:/dev/dri \
  rocm/dev-ubuntu-22.04:latest \
  clinfo
```

## Troubleshooting Common Issues

If the GPU is not visible inside the container, check these common problems:

```bash
# Check if the device files exist on the host
ls -la /dev/dri/ /dev/nvidia* 2>/dev/null

# Verify the user has permission to access GPU devices
groups $USER
# Should include 'video' and/or 'render'

# Check if SELinux is blocking access (on Fedora/RHEL)
sudo ausearch -m avc -ts recent | grep nvidia
# If SELinux is blocking, create a policy or set permissive mode for testing
sudo setsebool -P container_use_devices on

# For rootless podman, check subuid/subgid mapping
podman unshare cat /proc/self/uid_map
```

## Conclusion

GPU passthrough with Podman is a powerful capability that brings hardware acceleration into containerized workflows. By using device file mapping or the CDI specification, you can give containers direct access to GPU resources with minimal overhead. Start with the `--device` flag for simple setups, and consider the NVIDIA Container Toolkit and CDI for production environments where you need automated device discovery and configuration. With rootless support and strong security defaults, Podman makes GPU-accelerated containers both accessible and secure.
