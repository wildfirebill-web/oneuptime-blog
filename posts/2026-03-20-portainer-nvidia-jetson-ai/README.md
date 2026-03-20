# How to Install Portainer on NVIDIA Jetson for AI Edge Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, NVIDIA Jetson, AI, Edge Computing, Docker, ARM64, Self-Hosted

Description: Deploy Portainer on NVIDIA Jetson devices to manage GPU-accelerated AI containers at the edge with full CUDA support and visual management.

## Introduction

NVIDIA Jetson devices (Nano, Xavier NX, AGX Xavier, AGX Orin) are purpose-built for AI inference at the edge. With CUDA support, TensorRT, and NVIDIA's container runtime, Jetson devices can run GPU-accelerated containers. Portainer provides a management interface for deploying and managing these AI workloads.

## Prerequisites

- NVIDIA Jetson device with JetPack 5.x or 6.x installed
- At least 8GB storage (NVMe SSD recommended)
- SSH access enabled
- Docker (included with JetPack)

## Step 1: Verify JetPack Installation

```bash
# Check JetPack version
cat /etc/nv_tegra_release

# Verify CUDA is available
nvcc --version

# Check available GPU memory
sudo tegrastats
```

## Step 2: Configure NVIDIA Container Runtime

JetPack includes the NVIDIA container runtime, but verify Docker is configured to use it:

```bash
# Verify NVIDIA runtime is registered
docker info | grep Runtimes

# Should show: nvidia runc

# If not, configure it
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

## Step 3: Test GPU Access in Containers

```bash
# Test that containers can access the GPU
docker run --rm --runtime=nvidia \
  nvcr.io/nvidia/l4t-base:r32.7.1 \
  nvidia-smi

# Or use the newer l4t-jetpack image
docker run --rm --runtime=nvidia \
  nvcr.io/nvidia/l4t-jetpack:r36.2.0 \
  python3 -c "import torch; print(torch.cuda.is_available())"
```

## Step 4: Install Portainer

```bash
# Create data volume
docker volume create portainer_data

# Deploy Portainer (ARM64 compatible)
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Deploy AI Containers via Portainer

### Example: TensorRT Inference Container

In Portainer, create a new stack named `ai-inference`:

```yaml
version: "3.8"

services:
  # TensorRT inference server
  triton:
    image: nvcr.io/nvidia/tritonserver:23.10-py3
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    ports:
      - "8000:8000"   # HTTP inference endpoint
      - "8001:8001"   # gRPC inference endpoint
      - "8002:8002"   # Metrics endpoint
    volumes:
      # Model repository
      - /data/models:/models
    command: >
      tritonserver
      --model-repository=/models
      --allow-metrics=true
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### Example: Object Detection Container

```yaml
version: "3.8"

services:
  # YOLOv8 detection service
  yolo-service:
    image: ultralytics/ultralytics:latest-jetson-jetpack5
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    ports:
      - "5000:5000"
    volumes:
      - /data/yolo-models:/models
      - /dev/video0:/dev/video0  # Camera access
    devices:
      - /dev/video0:/dev/video0
    restart: unless-stopped
```

## Step 6: Monitor GPU Usage

Deploy a GPU monitoring container via Portainer:

```yaml
version: "3.8"

services:
  # Jetson stats exporter for Prometheus
  jetson-stats:
    image: ajeetraina/jetson-stats-exporter:latest
    runtime: nvidia
    privileged: true
    ports:
      - "9545:9545"
    volumes:
      - /run/jtop.sock:/run/jtop.sock
    restart: unless-stopped
```

Install jtop on the host:

```bash
sudo pip3 install -U jetson-stats
sudo systemctl restart jtop
```

## Jetson-Specific Portainer Tips

### Power Mode Management

Set the optimal power mode for your workload:

```bash
# List power modes
sudo nvpmodel -q

# Set to MAX performance (mode 0)
sudo nvpmodel -m 0

# Maximize clocks
sudo jetson_clocks
```

### Using Portainer to Manage Model Updates

Create a Portainer webhook for CI/CD model deployment:

1. In Portainer, navigate to **Stacks**
2. Click on your AI stack
3. Enable **GitOps updates** and configure webhook
4. Trigger the webhook from your CI pipeline when a new model is ready

## Conclusion

NVIDIA Jetson devices with Portainer provide a powerful platform for AI edge deployments. Portainer's stack management simplifies deploying complex multi-container AI pipelines, while the NVIDIA container runtime ensures GPU access is properly configured for each container. The visual management interface makes it easy to monitor GPU-accelerated workloads without needing to SSH into the device for routine operations.
