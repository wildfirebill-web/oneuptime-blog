# How to Fix GPU Enabling Errors in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, GPU, Docker, NVIDIA, CUDA, Container Configuration

Description: Learn how to fix GPU enabling errors in Portainer when deploying containers that require NVIDIA or AMD GPU access, including runtime configuration and capability settings.

---

GPU passthrough to Docker containers requires the NVIDIA Container Toolkit (for NVIDIA GPUs) or ROCm drivers (for AMD GPUs) on the host. Without these, Portainer will surface Docker daemon errors when you enable GPU access in the container settings.

## Prerequisites Check

```bash
# Verify NVIDIA drivers are installed
nvidia-smi

# Verify NVIDIA Container Toolkit is installed
nvidia-ctk --version

# Verify Docker can use the NVIDIA runtime
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi
```

## Error: "could not select device driver with capabilities: [[gpu]]"

This means the NVIDIA Container Toolkit is not installed or not configured:

```bash
# Install NVIDIA Container Toolkit on Ubuntu
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

# Configure Docker to use the NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

## Error: "unknown runtime specified nvidia"

The NVIDIA runtime is not registered with Docker:

```bash
# Check /etc/docker/daemon.json
cat /etc/docker/daemon.json

# It should contain:
{
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-runtime": "nvidia"
}

# If missing, add it and restart Docker
sudo systemctl restart docker
```

## Configuring GPU in Portainer

In Portainer's container creation UI, the GPU option requires deploying the container with specific runtime settings. Use a stack instead for reliable GPU configuration:

```yaml
version: "3.8"

services:
  gpu-app:
    image: nvidia/cuda:12.0-base
    # Request all available GPUs
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    command: nvidia-smi
```

## AMD GPU Support

For AMD GPUs, use the ROCm runtime:

```yaml
services:
  rocm-app:
    image: rocm/pytorch:latest
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    group_add:
      - video
      - render
```
