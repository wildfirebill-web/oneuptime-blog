# How to Enable GPU Support for Containers in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Container, GPU, Machine Learning

Description: Learn how to enable GPU access for Docker containers in Portainer using NVIDIA Container Toolkit for machine learning and compute workloads.

## Introduction

GPU-accelerated containers are essential for machine learning inference, model training, video transcoding, and scientific computing. Portainer supports configuring GPU access through the NVIDIA Container Toolkit. This guide covers the setup from host configuration to deploying GPU-enabled containers.

## Prerequisites

- NVIDIA GPU on the Docker host
- NVIDIA drivers installed on the host
- Docker Engine installed
- Portainer installed and connected to the GPU-enabled host

## Step 1: Install NVIDIA Container Toolkit on the Host

Before configuring in Portainer, the host must have the NVIDIA Container Toolkit installed:

```bash
# Install NVIDIA Container Toolkit (Ubuntu/Debian)

# Add NVIDIA package repositories

distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
    sudo apt-key add -
curl -s -L "https://nvidia.github.io/nvidia-docker/${distribution}/nvidia-docker.list" | \
    sudo tee /etc/apt/sources.list.d/nvidia-docker.list

# Install the toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Configure Docker to use the NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker

# Restart Docker
sudo systemctl restart docker
```

```bash
# For RHEL/CentOS/Fedora:
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/${distribution}/nvidia-docker.repo | \
    sudo tee /etc/yum.repos.d/nvidia-docker.repo

sudo yum install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

## Step 2: Verify GPU Access Works

```bash
# Test GPU access from a container (without Portainer first):
docker run --rm --gpus all nvidia/cuda:12.3.0-base-ubuntu22.04 nvidia-smi

# Expected output (example):
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 545.23.08    Driver Version: 545.23.08    CUDA Version: 12.3    |
|-------------------------------|----------------------|----------------------+
| GPU  Name        Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|===============================|======================|======================|
|   0  NVIDIA A100-SXM...    On  | 00000000:00:04.0 Off |                    0 |
| N/A   30C    P0    55W / 400W  |      0MiB / 40960MiB |      0%      Default |
+-----------------------------------------------------------------------------+
```

## Step 3: Enable GPU in Portainer (Docker Compose / Stack)

In Portainer, configure GPU access via the **Deploy resources** section in a stack:

```yaml
# gpu-workload-stack.yml
version: "3.8"

services:
  # ML inference service with GPU access
  inference:
    image: myorg/ml-inference:cuda12
    restart: unless-stopped
    shm_size: '8g'   # Large shared memory for ML workloads
    deploy:
      resources:
        reservations:
          devices:
            # Reserve all available GPUs
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - CUDA_VISIBLE_DEVICES=0
      - MODEL_PATH=/models/my-model
    volumes:
      - model_data:/models

  # Training job (uses specific GPU)
  trainer:
    image: pytorch/pytorch:2.2.0-cuda12.1-cudnn8-devel
    restart: "no"   # One-time training job
    shm_size: '16g'
    deploy:
      resources:
        reservations:
          devices:
            # Reserve a specific GPU by index
            - driver: nvidia
              device_ids: ["0"]   # GPU 0 only
              capabilities: [gpu, compute, utility]
    command: python train.py --epochs 100 --batch-size 32

volumes:
  model_data:
```

## Step 4: Enable GPU for Individual Containers

For a single container (not a stack), Portainer BE allows GPU configuration in the container creation form:

1. Navigate to **Containers > Add container**.
2. Scroll to **Runtime & Resources**.
3. Find the **GPU** section.
4. Enable **Use all GPUs** or specify individual GPU IDs.

The equivalent Docker CLI:

```bash
# Access all GPUs:
docker run --gpus all \
  -e CUDA_VISIBLE_DEVICES=all \
  myorg/ml-app:latest

# Access specific GPU by index:
docker run --gpus '"device=0"' \
  myorg/ml-app:latest

# Access two specific GPUs:
docker run --gpus '"device=0,1"' \
  myorg/ml-app:latest

# Access GPU with specific capabilities:
docker run --gpus 'all,"capabilities=compute,utility"' \
  myorg/ml-app:latest
```

## Step 5: Common GPU Workloads

### Ollama (Local LLM Inference)

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### Stable Diffusion

```yaml
services:
  stable-diffusion:
    image: ghcr.io/automatic1111/stable-diffusion-webui:latest
    restart: unless-stopped
    ports:
      - "7860:7860"
    volumes:
      - sd_models:/app/models
    shm_size: '8g'
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### Transcoding with NVENC

```yaml
services:
  ffmpeg-transcoder:
    image: jrottenberg/ffmpeg:5.1-nvidia
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [gpu, video]  # Enable video encode/decode
```

## Step 6: Monitor GPU Usage

Inside the container:

```bash
# Via Portainer console:
nvidia-smi   # Static GPU info
watch -n 1 nvidia-smi   # Live GPU monitoring

# GPU memory usage:
nvidia-smi --query-gpu=memory.used,memory.free --format=csv
```

On the host:

```bash
# Monitor GPU usage across all containers:
nvidia-smi dmon -s u   # Utilization monitoring
```

## Troubleshooting GPU Access

```bash
# Error: "could not select device driver "nvidia" with capabilities: [[gpu]]"
# Fix: ensure NVIDIA Container Toolkit is installed and configured
nvidia-ctk runtime configure --runtime=docker && systemctl restart docker

# Error: "Failed to initialize NVML"
# Fix: check NVIDIA driver installation
nvidia-smi   # Should work on the host

# Error: GPU not visible inside container
# Fix: check CUDA_VISIBLE_DEVICES env var
# Set to "all" or specific GPU index
```

## Conclusion

Enabling GPU support for containers in Portainer requires proper host setup (NVIDIA drivers + Container Toolkit) followed by configuring GPU resources in the container or stack definition. Once set up, Portainer makes it straightforward to deploy and manage GPU-accelerated workloads - from machine learning inference to video transcoding - as part of your standard container management workflow.
