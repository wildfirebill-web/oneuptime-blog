# How to Create a K3s Cluster in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, K3s, Cluster Management

Description: Learn how to create a lightweight K3s Kubernetes cluster through Rancher for edge, IoT, and resource-constrained environments.

K3s is a lightweight, certified Kubernetes distribution designed for edge computing, IoT, and resource-constrained environments. It uses about half the memory of standard Kubernetes and ships as a single binary. This guide shows you how to create a K3s cluster through the Rancher management UI.

## Prerequisites

- A running Rancher installation (v2.7 or later)
- At least one Linux node with:
  - Minimum 1 GB RAM and 1 CPU (2 GB RAM and 2 CPUs recommended)
  - A supported OS (Ubuntu 20.04+, RHEL 8+, Debian 11+, SLES 15+)
  - Systemd or openrc init system
- Network connectivity between nodes and to the Rancher server

## Step 1: Prepare the Nodes

K3s has minimal requirements compared to standard Kubernetes distributions.

### Basic Node Setup

```bash
# Disable swap (optional for K3s but recommended)
sudo swapoff -a

# Ensure required kernel modules are loaded
sudo modprobe br_netfilter
sudo modprobe overlay

# Set up sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k3s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Required Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API |
| 8472 | UDP | Flannel VXLAN |
| 10250 | TCP | kubelet |
| 2379-2380 | TCP | etcd (HA mode) |

## Step 2: Create the Cluster in Rancher

1. Log in to the Rancher UI
2. Go to **Cluster Management**
3. Click **Create**
4. Under **Use existing nodes and create a cluster using RKE2/K3s**, select **Custom**

## Step 3: Configure Cluster Settings

### Basic Settings

- **Cluster Name**: Enter a name (e.g., `edge-k3s-cluster`)
- **Kubernetes Version**: Select a K3s version from the dropdown

Make sure you select a K3s version (these typically show with a `+k3s` suffix).

### Network Configuration

Select the CNI:

- **Flannel** (default for K3s): Simple overlay networking
- **Canal**: Flannel + Calico network policies
- **Calico**: Full Calico implementation

### Service CIDR and Cluster CIDR

Configure the network ranges if the defaults conflict with your network:

```
Cluster CIDR: 10.42.0.0/16
Service CIDR: 10.43.0.0/16
Cluster DNS: 10.43.0.10
```

## Step 4: Configure K3s-Specific Options

### Datastore

K3s supports two datastore options:

- **Embedded etcd** (default for HA): Built-in etcd for multi-server setups
- **Embedded SQLite**: Single-server only, minimal resource usage

For production with HA, use embedded etcd with 3 server nodes.

### Disable Components

K3s bundles several components that you can disable if not needed:

```yaml
# In the additional server args
disable:
  - traefik      # Disable built-in Traefik ingress
  - servicelb    # Disable built-in ServiceLB
  - metrics-server  # If using external metrics
```

Configure these through the Rancher UI under advanced cluster options.

### Registry Mirror

For air-gapped or private registry environments:

```yaml
mirrors:
  docker.io:
    endpoint:
      - https://registry.yourdomain.com
  "registry.yourdomain.com":
    rewrite:
      "^rancher/(.*)": "mirror/rancher/$1"
```

## Step 5: Register Nodes

### Register Server Nodes

Select **etcd** and **Control Plane** roles and copy the registration command:

```bash
curl -fL https://rancher.yourdomain.com/system-agent-install.sh | sudo sh -s - \
  --server https://rancher.yourdomain.com \
  --label 'cattle.io/os=linux' \
  --token <TOKEN> \
  --etcd --controlplane
```

For a single-server setup, run this on one node. For HA, run on 3 nodes.

### Register Agent (Worker) Nodes

Select only the **Worker** role:

```bash
curl -fL https://rancher.yourdomain.com/system-agent-install.sh | sudo sh -s - \
  --server https://rancher.yourdomain.com \
  --label 'cattle.io/os=linux' \
  --token <TOKEN> \
  --worker
```

### Edge Device Registration

For edge devices with limited resources, you can add labels during registration:

```bash
curl -fL https://rancher.yourdomain.com/system-agent-install.sh | sudo sh -s - \
  --server https://rancher.yourdomain.com \
  --label 'cattle.io/os=linux' \
  --label 'location=warehouse-1' \
  --label 'device-type=gateway' \
  --token <TOKEN> \
  --worker
```

## Step 6: Monitor Provisioning

Watch the cluster come online:

1. In the Rancher UI, navigate to your cluster
2. Watch nodes register and roles get assigned
3. Wait for status to change to `Active`

Check system agent logs on nodes:

```bash
sudo journalctl -u rancher-system-agent -f
```

Check K3s service logs:

```bash
sudo journalctl -u k3s -f          # Server nodes
sudo journalctl -u k3s-agent -f    # Agent nodes
```

## Step 7: Verify the Cluster

Once the cluster is Active:

```bash
# From Rancher UI, download kubeconfig
kubectl get nodes -o wide
kubectl get pods -A
```

K3s system pods include:

- `coredns`
- `local-path-provisioner` (default storage)
- `metrics-server`
- `traefik` (if not disabled)
- Network plugin pods (flannel, canal, or calico)

Check resource usage to confirm K3s's lightweight footprint:

```bash
kubectl top nodes
kubectl top pods -A
```

## Step 8: Deploy Workloads

Test the cluster with a simple deployment:

```bash
kubectl create deployment hello --image=nginx --replicas=2
kubectl expose deployment hello --port=80 --type=NodePort
kubectl get svc hello
```

For edge use cases, deploy lightweight applications:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-sensor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edge-sensor
  template:
    metadata:
      labels:
        app: edge-sensor
    spec:
      containers:
      - name: sensor
        image: your-registry/edge-sensor:latest
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
```

## Use Cases for K3s in Rancher

- **Edge computing**: Deploy K3s on edge devices and manage them centrally from Rancher
- **Development environments**: Quick, lightweight clusters for development and testing
- **IoT gateways**: Run Kubernetes workloads on IoT gateway devices
- **CI/CD runners**: Ephemeral clusters for build and test pipelines
- **Branch offices**: Small clusters at remote locations managed centrally

## Troubleshooting

- **Node not joining**: Verify network connectivity to Rancher. Check system-agent logs.
- **High memory usage**: K3s should use 500-700 MB on server nodes. If higher, check for memory leaks in workloads.
- **Storage issues**: K3s uses local-path-provisioner by default. For production, consider adding a more robust storage solution.
- **DNS resolution failures**: Check coredns pods and node DNS configuration.

## Conclusion

K3s clusters created through Rancher give you the best of both worlds: a lightweight Kubernetes distribution perfect for edge and resource-constrained environments, managed through Rancher's comprehensive UI. The setup process is fast, the resource footprint is small, and Rancher provides the management layer needed to operate K3s clusters at scale across distributed locations.
