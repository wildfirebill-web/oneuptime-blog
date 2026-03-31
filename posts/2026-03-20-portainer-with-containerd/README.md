# How to Use Portainer with Containerd (Without Docker)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Containerd, Kubernetes, CRI, Container

Description: Deploy and configure Portainer to manage containers running directly on containerd without requiring the Docker daemon.

## Introduction

Containerd is a high-performance container runtime originally extracted from Docker and now a graduated CNCF project. Many Kubernetes distributions (k3s, RKE2, EKS) use containerd directly. Portainer can manage containerd-based environments via its Kubernetes integration or via an agent.

## Understanding the Architecture

When Docker is absent, Portainer connects to containerd via:
1. **Kubernetes API** (recommended) - Portainer manages pods/deployments via the K8s API
2. **Portainer Agent** - Deploy the agent as a containerd task
3. **nerdctl** compatibility - nerdctl provides Docker-compatible CLI for containerd

## Method 1: Managing Containerd via Kubernetes (Recommended)

Most containerd deployments run inside Kubernetes. Use Portainer's Kubernetes environment type:

```bash
# Install k3s (uses containerd by default)

curl -sfL https://get.k3s.io | sh -

# Verify containerd is the runtime
sudo k3s crictl info | grep runtimeType

# Get the kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml
```

Add to Portainer:
- Environment type: **Kubernetes**
- URL: `https://your-k3s-host:6443`
- Import the kubeconfig file

## Method 2: Using the Portainer Agent with Containerd

```bash
# Install nerdctl (Docker-compatible CLI for containerd)
NERDCTL_VERSION=1.7.0
wget https://github.com/containerd/nerdctl/releases/download/v${NERDCTL_VERSION}/nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz
sudo tar xzf nerdctl-${NERDCTL_VERSION}-linux-amd64.tar.gz -C /usr/local/bin/

# Install CNI plugins for networking
sudo mkdir -p /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
sudo tar xzf cni-plugins-linux-amd64-v1.4.0.tgz -C /opt/cni/bin/

# Verify nerdctl works with containerd
sudo nerdctl ps
sudo nerdctl info
```

## Setting Up containerd

```bash
# Install containerd
sudo apt-get update && sudo apt-get install -y containerd

# Generate default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable and start containerd
sudo systemctl enable --now containerd

# Verify
sudo ctr version
```

## Running Portainer Agent via nerdctl

```bash
# Create the Portainer agent network
sudo nerdctl network create portainer_agent_network

# Run Portainer agent
sudo nerdctl run -d \
  --name portainer_agent \
  --network portainer_agent_network \
  -p 9001:9001 \
  --restart=always \
  -v /run/containerd/containerd.sock:/var/run/docker.sock \
  -v /var/lib/containerd:/var/lib/docker \
  -v /:/host \
  portainer/agent:latest

# Check agent is running
sudo nerdctl ps
```

## Deploying Portainer Server

If running Portainer server on the same host:

```bash
# Create volume for Portainer data
sudo nerdctl volume create portainer_data

# Run Portainer
sudo nerdctl run -d \
  --name portainer \
  --restart=always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /run/containerd/containerd.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Using crictl for Debugging

crictl is a CLI tool for container runtimes implementing the CRI spec:

```bash
# Install crictl
VERSION="v1.29.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin/

# Configure crictl to use containerd
sudo tee /etc/crictl.yaml << 'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# List running pods/containers
sudo crictl pods
sudo crictl ps

# Inspect a container
sudo crictl inspect <container-id>
```

## Containerd Namespaces

Containerd uses namespaces to isolate resources:

```bash
# List all namespaces
sudo ctr namespaces list

# Kubernetes uses the 'k8s.io' namespace
sudo ctr -n k8s.io containers list

# Default namespace for nerdctl
sudo ctr -n default containers list
```

## Conclusion

Portainer integrates well with containerd-based environments, primarily through the Kubernetes API or the Portainer agent. For direct containerd management, nerdctl provides Docker API compatibility. This setup is particularly relevant for Kubernetes distributions like k3s, RKE2, and managed Kubernetes services that use containerd as their container runtime.
