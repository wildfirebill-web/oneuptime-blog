# How to Set Up K3s with Rootless Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Security, Rootless, Linux, Container, DevOps

Description: Learn how to run K3s in rootless mode to improve security by running the entire cluster without root privileges.

## Introduction

Running container workloads as root is a security risk - if a container escape occurs, an attacker could gain root access to the host. K3s supports **rootless mode**, which runs the entire K3s server (including containerd and networking) as a non-root user using user namespaces. This significantly reduces the attack surface. This guide covers setting up and using K3s in rootless mode.

## Understanding Rootless Mode

In rootless mode:
- K3s runs as a regular user (no root required after initial setup)
- User namespaces isolate the K3s process from the root namespace
- rootlesskit handles networking through user-space networking
- Some features have limitations (no binding to ports < 1024 without special config)

## Prerequisites

- Linux kernel 5.11+ (for better cgroup v2 support)
- User namespaces enabled
- `newuidmap` and `newgidmap` tools (`uidmap` package)
- A non-root user account
- systemd user session support (optional but recommended)

## Step 1: Verify System Requirements

```bash
# Check if user namespaces are enabled

cat /proc/sys/kernel/unprivileged_userns_clone
# Should be 1

# If 0, enable it
echo 1 > /proc/sys/kernel/unprivileged_userns_clone
echo "kernel.unprivileged_userns_clone=1" >> /etc/sysctl.conf
sysctl -p

# Check for required tools
which newuidmap newgidmap || apt-get install -y uidmap

# Check if cgroup v2 is available (recommended)
stat -fc %T /sys/fs/cgroup/
# 'cgroup2fs' = v2 (good), 'tmpfs' = v1

# Verify user has a subuid/subgid range
grep $USER /etc/subuid /etc/subgid
# Should show: username:100000:65536
# If missing:
usermod --add-subuids 100000-165535 $USER
usermod --add-subgids 100000-165535 $USER
```

## Step 2: Install K3s in Rootless Mode

Switch to the non-root user you'll run K3s as:

```bash
# Create a dedicated user (or use existing non-root user)
useradd -m -s /bin/bash k3s-user
su - k3s-user

# Install K3s in rootless mode
curl -sfL https://get.k3s.io | sh -s - server \
  --rootless

# Alternatively, with specific options
curl -sfL https://get.k3s.io | sh -s - \
  --rootless \
  --disable traefik
```

## Step 3: Configure systemd User Service

For rootless K3s to start automatically:

```bash
# As the k3s-user, enable lingering (allows user systemd to run at boot)
# Run this as root:
loginctl enable-linger k3s-user

# Switch back to k3s-user
su - k3s-user

# Check if user systemd service is installed
systemctl --user status k3s

# Start K3s as user service
systemctl --user start k3s

# Enable auto-start
systemctl --user enable k3s

# Check status
systemctl --user status k3s

# View logs
journalctl --user -u k3s -f
```

## Step 4: Configure kubectl for Rootless K3s

```bash
# As k3s-user, set the kubeconfig path
export KUBECONFIG=~/.kube/k3s.yaml

# K3s rootless kubeconfig is at:
ls ~/.kube/
# k3s.yaml

# Or use the XDG_RUNTIME_DIR path
ls $XDG_RUNTIME_DIR/k3s/

# Add to .bashrc for persistence
echo 'export KUBECONFIG=~/.kube/k3s.yaml' >> ~/.bashrc
source ~/.bashrc

# Test
kubectl get nodes
```

## Step 5: Configure Networking for Rootless Mode

Rootless mode uses user-space networking with limitations:

```bash
# Check rootlesskit is running
ps aux | grep rootlesskit

# Rootless K3s uses a different network port mapping approach
# By default, pods can't bind to ports < 1024

# For port access, use NodePort (>1024) or configure port forwarding:

# Option 1: Use higher ports
# NodePort services use ports 30000-32767 by default

# Option 2: Allow binding to privileged ports via sysctl
echo 0 > /proc/sys/net/ipv4/ip_unprivileged_port_start
# Or:
echo "net.ipv4.ip_unprivileged_port_start=0" >> /etc/sysctl.conf
sysctl -p

# Option 3: Use capabilities (as root, grant to rootlesskit)
setcap 'cap_net_bind_service=+ep' /usr/local/bin/rootlesskit
```

## Step 6: Verify Rootless Mode is Working

```bash
# Check K3s is running as non-root
ps aux | grep k3s | head -5
# The k3s process should show as k3s-user, not root

# Verify no root processes
pgrep -u root k3s || echo "No K3s root processes"

# Check inside a container
kubectl run whoami --image=busybox --restart=Never -- \
  sh -c 'id && cat /proc/self/status | grep -E "Uid|Gid|CapE"'

kubectl logs whoami
kubectl delete pod whoami

# Verify namespaces
kubectl run ns-check --image=busybox --restart=Never -- \
  sh -c 'ls /proc/1/ns/'
```

## Step 7: Storage Considerations

Rootless mode affects how volumes work:

```bash
# Check K3s data directory in rootless mode
ls -la ~/.local/share/rancher/k3s/
# Data is stored in user home directory, not /var/lib/rancher/k3s/

# Configure local-path-provisioner for rootless
# The default storage path changes to ~/local-path-provisioner/
kubectl get storageclass
kubectl describe sc local-path | grep path

# Custom storage path
kubectl edit configmap local-path-config -n kube-system
# Update helperPod.nodePathMap to use user-accessible path
```

## Step 8: Deploy a Test Workload

```yaml
# rootless-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rootless-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rootless-nginx
  template:
    metadata:
      labels:
        app: rootless-nginx
    spec:
      # Rootless containers should not run as root
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: nginx
          # Use nginx-unprivileged which runs as non-root
          image: nginxinc/nginx-unprivileged:alpine
          ports:
            - containerPort: 8080  # Non-privileged port
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
```

```bash
kubectl apply -f rootless-test.yaml
kubectl get pods
kubectl port-forward deployment/rootless-nginx 8080:8080 &
curl http://localhost:8080/
```

## Step 9: Limitations of Rootless Mode

Be aware of these limitations:

```bash
# 1. No privileged containers (unless sysctl is configured)
# 2. Port binding below 1024 requires extra config
# 3. Some CNI features may not work
# 4. Node exporter requires privileged for some metrics

# Check which K3s features are limited
k3s server --rootless --help 2>&1 | grep -i warning

# Some features that require root:
# - hostNetwork: true with ports < 1024
# - privileged: true containers
# - Direct device access (/dev/*) without userspace mapping
```

## Conclusion

K3s rootless mode provides a significant security improvement by eliminating root privileges from the container orchestration stack. While it has some limitations around port binding and privileged containers, rootless mode is excellent for development environments, multi-tenant systems, or any deployment where security is paramount. As Linux kernel support for user namespaces continues to improve, rootless Kubernetes deployments will become more capable and widely adopted.
