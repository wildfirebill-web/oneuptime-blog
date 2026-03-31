# How to Install K3s on a Single Node

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Rancher, Installation, Single Node

Description: A beginner-friendly guide to installing K3s on a single Linux node and running your first workload in minutes.

## Introduction

K3s is a lightweight, certified Kubernetes distribution from Rancher designed for resource-constrained environments. A single K3s binary contains everything needed to run a full Kubernetes cluster - API server, scheduler, controller manager, kubelet, and kube-proxy - making it perfect for development, edge deployments, and single-node production workloads.

## System Requirements

- **OS**: Linux (Ubuntu 20.04+, Debian 11+, CentOS 7+, RHEL 7+)
- **CPU**: 1 vCPU minimum (2+ recommended)
- **RAM**: 512 MB minimum (1 GB+ recommended)
- **Disk**: 1 GB minimum for K3s components
- **Architecture**: x86_64, ARM64, or ARMv7

## Step 1: Run the K3s Install Script

K3s provides a simple one-liner installation script:

```bash
# Install K3s using the official install script

curl -sfL https://get.k3s.io | sudo sh -

# The script automatically:
# - Downloads the K3s binary
# - Creates a systemd service
# - Starts the K3s server
# - Installs kubectl, crictl, and ctr
```

### Installing a Specific Version

```bash
# Install a specific K3s version
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_VERSION="v1.28.7+k3s1" \
    sudo sh -
```

### Installing Without Starting Immediately

```bash
# Install but don't start the service
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_SKIP_START=true \
    sudo sh -
```

## Step 2: Verify the Installation

```bash
# Check the K3s service status
sudo systemctl status k3s

# View the K3s server logs
sudo journalctl -u k3s -n 50 --no-pager

# Wait for the node to be ready
sudo k3s kubectl wait --for=condition=Ready node --all --timeout=120s
```

## Step 3: Configure kubectl

K3s installs `kubectl` as a wrapper. You can use it directly:

```bash
# Use kubectl via k3s
sudo k3s kubectl get nodes
sudo k3s kubectl get pods --all-namespaces

# Or set up kubectl to work without sudo
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Now use kubectl normally
kubectl get nodes
kubectl get pods --all-namespaces
```

## Step 4: Verify Cluster Components

```bash
# Check that all system pods are running
kubectl get pods -n kube-system

# K3s includes the following by default:
# - CoreDNS
# - Traefik Ingress Controller
# - ServiceLB (Klipper)
# - Metrics Server
# - Local Path Provisioner

# Check running services
kubectl get svc -n kube-system
```

## Step 5: Deploy Your First Application

```bash
# Deploy an nginx web server
kubectl create deployment nginx --image=nginx

# Wait for it to be ready
kubectl rollout status deployment/nginx

# Expose it on port 80
kubectl expose deployment nginx --port=80 --type=NodePort

# Get the assigned NodePort
kubectl get svc nginx

# Test the application
NODE_PORT=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
curl http://localhost:$NODE_PORT
```

## Step 6: Deploy with an Ingress

K3s includes Traefik, which handles ingress routing:

```yaml
# nginx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: nginx.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

```bash
kubectl apply -f nginx-ingress.yaml

# Access via the hostname (add to /etc/hosts if testing locally)
echo "127.0.0.1 nginx.example.com" | sudo tee -a /etc/hosts
curl http://nginx.example.com
```

## Configuration Options

### Configure a Custom CIDR

```bash
# Install K3s with a custom pod CIDR
curl -sfL https://get.k3s.io | sudo sh -s - \
    --cluster-cidr=10.244.0.0/16 \
    --service-cidr=10.245.0.0/16
```

### Configure with a Config File

```bash
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
# K3s server configuration
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
cluster-dns: "10.43.0.10"
tls-san:
  - 192.168.1.100
  - k3s.example.com
disable:
  - traefik   # Disable Traefik if using a different ingress
EOF

# Install K3s (it will use the config file automatically)
curl -sfL https://get.k3s.io | sudo sh -
```

## Accessing the K3s Dashboard

```bash
# Deploy the Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create an admin token
kubectl -n kubernetes-dashboard create token admin-user

# Port-forward to the dashboard
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard 8443:443 &

# Open: https://localhost:8443
```

## Uninstalling K3s

```bash
# K3s installs an uninstall script
sudo /usr/local/bin/k3s-uninstall.sh
```

## Conclusion

Installing K3s on a single node takes less than a minute and gives you a fully functional Kubernetes environment. The built-in Traefik ingress, local path provisioner, and metrics server mean you can immediately deploy and expose applications without additional setup. K3s's single-binary design and minimal resource requirements make it the go-to choice for developers, edge deployments, and lightweight production workloads.
