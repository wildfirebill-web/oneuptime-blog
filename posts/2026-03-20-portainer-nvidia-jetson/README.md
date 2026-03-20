# How to Install Portainer on NVIDIA Jetson for AI Edge Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, NVIDIA Jetson, Edge AI, ARM, Docker

Description: Learn how to install Portainer on NVIDIA Jetson devices to manage AI and machine learning container deployments at the edge, with GPU access for inference workloads.

## Why Run Portainer on Jetson?

NVIDIA Jetson (Nano, Orin, AGX Xavier) devices are purpose-built for AI inference at the edge. Portainer on Jetson lets you:

- Deploy AI models as containers without SSH
- Manage multiple Jetson devices from a central Portainer Server
- Monitor GPU and CPU metrics for inference workloads
- Update models and applications remotely

## Jetson Architecture Notes

Jetson devices use ARM64 (aarch64) architecture. Portainer provides ARM64 images, and most AI frameworks (PyTorch, TensorFlow, ONNX) have Jetson-specific builds from NVIDIA NGC.

## Prerequisites

- NVIDIA JetPack 5.x or 6.x installed (includes Docker)
- Sufficient storage for Docker images (AI models can be large)
- Network connectivity

## Step 1: Verify Docker Is Running on Jetson

```bash
# JetPack includes Docker - verify it's active

sudo systemctl status docker

# Check Docker supports NVIDIA runtime
docker info | grep -i runtime
# Should include: Runtimes: nvidia runc

# Test GPU access in Docker
docker run --rm --runtime nvidia --gpus all \
  nvcr.io/nvidia/l4t-base:r36.2.0 \
  nvidia-smi
```

## Step 2: Install Portainer

```bash
# Create volume for Portainer data
docker volume create portainer_data

# Install Portainer CE (ARM64 image is auto-selected)
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Access Portainer at https://JETSON_IP:9443
```

## Step 3: Deploy an AI Inference Container via Portainer

In Portainer: **Stacks → Add Stack**

```yaml
version: "3.8"

services:
  inference:
    image: nvcr.io/nvidia/l4t-pytorch:r36.2.0-pth2.1-py3
    restart: unless-stopped
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=all
    volumes:
      - models:/models
      - /tmp:/tmp
    ports:
      - "8080:8080"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

volumes:
  models:
```

## Step 4: Install Portainer as an Edge Agent (Recommended for Multi-Device)

For managing multiple Jetson devices from a central Portainer Server:

```bash
# On the Jetson device, run the Edge Agent
# (get the exact command from Portainer Server > Environments > Add Edge Agent)
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -v /:/host \
  -v portainer_agent_data:/data \
  --restart always \
  -e EDGE=1 \
  -e EDGE_ID=<your-edge-id> \
  -e EDGE_KEY=<your-edge-key> \
  --name portainer_edge_agent \
  portainer/agent:latest
```

## Monitoring Jetson GPU Metrics

Deploy Prometheus + NVIDIA Jetson Stats Exporter:

```yaml
services:
  jetson-stats-exporter:
    image: atgroup09/prometheus-jetson-stats:latest
    privileged: true
    restart: unless-stopped
    ports:
      - "9101:9101"
```

## Jetson-Specific Considerations

| Consideration | Recommendation |
|--------------|----------------|
| Image size | AI images are 5-20GB; use an SSD |
| Power mode | Set max performance: `sudo nvpmodel -m 0` |
| Swap | Enable swap for memory-intensive models |
| Cooling | Ensure adequate cooling under sustained inference load |

## Conclusion

Portainer on NVIDIA Jetson provides a practical management layer for edge AI deployments. The combination of Portainer's container management with NVIDIA's GPU-accelerated Docker runtime lets you deploy, update, and monitor AI inference containers with the same workflows you use for any other containerized application.
