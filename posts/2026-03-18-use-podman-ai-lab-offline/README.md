# How to Use Podman AI Lab Offline

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, AI, Machine Learning, Offline, Air-Gapped, Privacy

Description: Learn how to set up and use Podman AI Lab in fully offline and air-gapped environments for maximum privacy and security.

---

> Running AI Lab offline ensures zero data leaves your machine, making it ideal for sensitive and classified workloads.

Many organizations operate in air-gapped or restricted network environments where internet access is not available. Podman AI Lab can run entirely offline once the models and container images are pre-downloaded. This guide covers preparing your system for offline AI operations, transferring assets, and running everything without network connectivity.

---

## Prerequisites

You need two machines for the initial setup:

- **Online machine**: Has internet access for downloading models and images.
- **Offline machine**: The target system with Podman installed but no internet.

```bash
# On the online machine, ensure Podman is installed

podman --version

# Create a staging directory for offline assets
mkdir -p ~/offline-ai-lab/{images,models}
```

## Step 1: Download Container Images on the Online Machine

```bash
# Pull all required container images
# The inference server
podman pull ghcr.io/ggerganov/llama.cpp:server
podman pull ghcr.io/containers/ai-lab-model-service:latest

# Save images as tar archives for transfer
podman save -o ~/offline-ai-lab/images/llama-server.tar \
  ghcr.io/ggerganov/llama.cpp:server

podman save -o ~/offline-ai-lab/images/ai-lab-model-service.tar \
  ghcr.io/containers/ai-lab-model-service:latest

# If you plan to use recipes, save those images too
podman pull ghcr.io/containers/ai-lab-recipe-chatbot:latest
podman save -o ~/offline-ai-lab/images/recipe-chatbot.tar \
  ghcr.io/containers/ai-lab-recipe-chatbot:latest

# Verify saved images
ls -lh ~/offline-ai-lab/images/
```

## Step 2: Download Models on the Online Machine

```bash
# Download model files in GGUF format
# Mistral 7B (general purpose)
curl -L -o ~/offline-ai-lab/models/mistral-7b-instruct-q4_k_m.gguf \
  "https://huggingface.co/TheBloke/Mistral-7B-Instruct-v0.2-GGUF/resolve/main/mistral-7b-instruct-v0.2.Q4_K_M.gguf"

# CodeLlama 7B (code generation)
curl -L -o ~/offline-ai-lab/models/codellama-7b-instruct-q4_k_m.gguf \
  "https://huggingface.co/TheBloke/CodeLlama-7B-Instruct-GGUF/resolve/main/codellama-7b-instruct.Q4_K_M.gguf"

# Verify downloads with checksums
sha256sum ~/offline-ai-lab/models/*.gguf > ~/offline-ai-lab/models/checksums.txt
cat ~/offline-ai-lab/models/checksums.txt

# Check total size of offline assets
du -sh ~/offline-ai-lab/
```

## Step 3: Transfer Assets to the Offline Machine

```bash
# Option 1: USB drive transfer
# Create a compressed archive
tar czf ~/offline-ai-lab.tar.gz -C ~/ offline-ai-lab/

# Copy to USB drive
sudo mount /dev/sdb1 /mnt/usb
cp ~/offline-ai-lab.tar.gz /mnt/usb/
sudo umount /mnt/usb

# On the offline machine, extract from USB
sudo mount /dev/sdb1 /mnt/usb
tar xzf /mnt/usb/offline-ai-lab.tar.gz -C ~/
sudo umount /mnt/usb

# Option 2: SCP over local network (if available)
scp -r ~/offline-ai-lab/ user@offline-machine:~/

# Verify checksums on the offline machine
cd ~/offline-ai-lab/models
sha256sum -c checksums.txt
```

## Step 4: Load Images on the Offline Machine

```bash
# Load all container images from tar archives
for tarfile in ~/offline-ai-lab/images/*.tar; do
  echo "Loading $(basename $tarfile)..."
  podman load -i "$tarfile"
done

# Verify images are available
podman images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Copy models to the expected location
mkdir -p ~/ai-models
cp ~/offline-ai-lab/models/*.gguf ~/ai-models/
ls -lh ~/ai-models/
```

## Step 5: Run AI Lab Offline

```bash
# Disable network for the container to enforce air-gap
podman run -d \
  --name offline-ai-server \
  --network none \
  -p 8080:8080 \
  -v ~/ai-models:/models:ro \
  ghcr.io/ggerganov/llama.cpp:server \
  --model /models/mistral-7b-instruct-q4_k_m.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  --ctx-size 4096 \
  --threads 4

# Wait for the server to start
podman logs -f offline-ai-server 2>&1 | grep -m1 "listening"

# Test locally (the server is only accessible from localhost)
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello, are you running offline?"}
    ],
    "max_tokens": 100
  }' | python3 -m json.tool
```

## Creating an Offline Startup Script

```bash
cat << 'SCRIPT' > ~/start-offline-ai.sh
#!/bin/bash
# Start the offline AI inference server
# This script runs entirely without network access

MODEL_DIR="${HOME}/ai-models"
MODEL_FILE="mistral-7b-instruct-q4_k_m.gguf"
CONTAINER_NAME="offline-ai-server"
PORT=8080

# Stop any existing instance
podman stop "$CONTAINER_NAME" 2>/dev/null
podman rm "$CONTAINER_NAME" 2>/dev/null

# Verify the model exists
if [ ! -f "${MODEL_DIR}/${MODEL_FILE}" ]; then
  echo "ERROR: Model not found at ${MODEL_DIR}/${MODEL_FILE}"
  exit 1
fi

# Start the server with no network access
podman run -d \
  --name "$CONTAINER_NAME" \
  --network none \
  -p ${PORT}:8080 \
  -v "${MODEL_DIR}:/models:ro" \
  ghcr.io/ggerganov/llama.cpp:server \
  --model "/models/${MODEL_FILE}" \
  --host 0.0.0.0 --port 8080 \
  --ctx-size 4096 --threads 4

echo "Offline AI server starting on port ${PORT}..."
echo "Test: curl http://localhost:${PORT}/health"
SCRIPT
chmod +x ~/start-offline-ai.sh
```

## Verifying Air-Gap Isolation

```bash
# Confirm the container has no network access
podman inspect offline-ai-server --format '{{.HostConfig.NetworkMode}}'
# Should output: none

# Attempt a network request from inside the container (should fail)
podman exec offline-ai-server curl -s http://example.com 2>&1 || echo "Network correctly blocked"

# Verify no DNS resolution is possible
podman exec offline-ai-server nslookup google.com 2>&1 || echo "DNS correctly blocked"
```

## Summary

Podman AI Lab works fully offline once you pre-download the necessary container images and model files. The process involves downloading assets on an internet-connected machine, transferring them to the target system, and loading them into Podman. Using the `--network none` flag ensures containers have zero network access, enforcing air-gap isolation. This setup is ideal for environments with strict security requirements where data must never leave the local machine.
