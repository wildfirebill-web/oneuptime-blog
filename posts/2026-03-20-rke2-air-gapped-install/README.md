# How to Install RKE2 in an Air-Gapped Environment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Air-Gapped, Installation, Security, Offline

Description: Learn how to install RKE2 in an air-gapped (offline) environment where nodes have no direct internet access, using pre-downloaded artifacts.

Air-gapped environments are isolated networks with no direct internet access, commonly used in government, defense, financial, and healthcare sectors for security reasons. Installing RKE2 in an air-gapped environment requires pre-downloading all necessary artifacts and setting up a private registry. This guide covers the complete air-gapped installation process.

## Prerequisites

- A machine with internet access for downloading artifacts
- A private container registry (Harbor, Nexus, or similar)
- A file server or shared storage accessible by all cluster nodes
- Linux nodes (Ubuntu, CentOS, Rocky Linux, etc.) with no internet access
- SSH access to all nodes

## Step 1: Download RKE2 Artifacts on an Internet-Connected Machine

```bash
# Set the RKE2 version to download

RKE2_VERSION="v1.28.8+rke2r1"

# Create a directory for artifacts
mkdir -p ~/rke2-artifacts && cd ~/rke2-artifacts

# Download the RKE2 installation script
curl -sfL https://get.rke2.io -o install.sh
chmod +x install.sh

# Download RKE2 binaries and checksum
# For amd64 (x86_64) systems:
curl -LO https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2.linux-amd64.tar.gz
curl -LO https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/sha256sum-amd64.txt

# Download the RKE2 images tarball
# This includes all container images needed by RKE2
curl -LO https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images.linux-amd64.tar.zst

# Verify checksum
sha256sum -c sha256sum-amd64.txt --ignore-missing

echo "All artifacts downloaded successfully"
ls -lh ~/rke2-artifacts/
```

## Step 2: Transfer Artifacts to Air-Gapped Nodes

```bash
# Copy artifacts to each server node
# Using scp (or rsync for large files)
for SERVER in server1 server2 server3; do
  echo "Copying to $SERVER..."
  ssh $SERVER "sudo mkdir -p /var/lib/rancher/rke2/agent/images/"

  # Copy installation script
  scp ~/rke2-artifacts/install.sh $SERVER:~/

  # Copy RKE2 binary tarball
  scp ~/rke2-artifacts/rke2.linux-amd64.tar.gz $SERVER:~/

  # Copy images (this is large - may take a while)
  scp ~/rke2-artifacts/rke2-images.linux-amd64.tar.zst \
    $SERVER:/var/lib/rancher/rke2/agent/images/
done

# For worker nodes
for WORKER in worker1 worker2 worker3; do
  echo "Copying to $WORKER..."
  ssh $WORKER "sudo mkdir -p /var/lib/rancher/rke2/agent/images/"
  scp ~/rke2-artifacts/install.sh $WORKER:~/
  scp ~/rke2-artifacts/rke2.linux-amd64.tar.gz $WORKER:~/
  scp ~/rke2-artifacts/rke2-images.linux-amd64.tar.zst \
    $WORKER:/var/lib/rancher/rke2/agent/images/
done
```

## Step 3: Install RKE2 on Air-Gapped Server Nodes

```bash
# On each server node - Run the installation in air-gapped mode
# The INSTALL_RKE2_ARTIFACT_PATH tells the installer to use local files

# Set the path to the artifacts
export INSTALL_RKE2_ARTIFACT_PATH=~/

# Install RKE2 server
INSTALL_RKE2_ARTIFACT_PATH=~/ sudo -E sh ~/install.sh

# The installer will:
# 1. Extract the RKE2 binary
# 2. Load the container images from the local tarball
# 3. Create systemd service files

echo "RKE2 installed from local artifacts"
```

## Step 4: Configure Private Registry

```bash
# Create registries.yaml to use a private registry mirror
sudo mkdir -p /etc/rancher/rke2/

cat <<EOF | sudo tee /etc/rancher/rke2/registries.yaml
mirrors:
  # Mirror Docker Hub through private registry
  "docker.io":
    endpoint:
    - "https://registry.internal.example.com"

  # Mirror ghcr.io
  "ghcr.io":
    endpoint:
    - "https://registry.internal.example.com"

  # Mirror quay.io
  "quay.io":
    endpoint:
    - "https://registry.internal.example.com"

configs:
  # TLS configuration for private registry
  "registry.internal.example.com":
    tls:
      ca_file: "/etc/ssl/certs/internal-ca.crt"
      # Or skip TLS verification (NOT recommended for production)
      # insecure_skip_verify: true
    auth:
      username: "rke2-puller"
      password: "registry-password"
EOF
```

## Step 5: Configure and Start RKE2 Server

```bash
# Create the RKE2 server configuration
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
# No internet access - use private registry
system-default-registry: registry.internal.example.com

# TLS SANs
tls-san:
  - "$(hostname -I | awk '{print $1}')"
  - "$(hostname -f)"

# Air-gapped specific settings
write-kubeconfig-mode: "0644"
EOF

# Start RKE2
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service

# Monitor startup (images load from local tarball)
sudo journalctl -u rke2-server -f
```

## Step 6: Install Helm Charts in Air-Gapped Mode

```bash
# Download Helm charts on internet-connected machine
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm pull rancher-latest/rancher --version 2.8.0

# Transfer to air-gapped environment
scp rancher-2.8.0.tgz server1:~/

# On the server, install from local chart
helm install rancher ~/rancher-2.8.0.tgz \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.internal.example.com \
  --set replicas=3 \
  --set bootstrapPassword=admin \
  --set useBundledSystemChart=true
```

## Step 7: Synchronize Images to Private Registry

```bash
# Script to sync required images to private registry
# Run on an internet-connected machine with access to the private registry

cat > sync-images.sh << 'EOF'
#!/bin/bash
PRIVATE_REGISTRY="registry.internal.example.com"
RKE2_VERSION="v1.28.8+rke2r1"

# Download the images list
curl -LO https://github.com/rancher/rke2/releases/download/${RKE2_VERSION}/rke2-images-all.linux-amd64.txt

# Pull and push each image
while IFS= read -r image; do
  echo "Syncing: $image"
  docker pull $image

  # Retag for private registry
  LOCAL_IMAGE="${PRIVATE_REGISTRY}/${image#*/}"
  docker tag $image $LOCAL_IMAGE
  docker push $LOCAL_IMAGE
done < rke2-images-all.linux-amd64.txt

echo "Image sync complete"
EOF

chmod +x sync-images.sh
```

## Conclusion

Installing RKE2 in an air-gapped environment requires careful preparation and planning, but the process is well-documented and reliable once you have all artifacts prepared. The key success factors are: pre-downloading all container images, setting up a private registry mirror for ongoing operations, and ensuring the RKE2 configuration points to your internal infrastructure. Air-gapped RKE2 clusters can be fully managed by an on-premises Rancher installation, providing the same capabilities as internet-connected deployments.
