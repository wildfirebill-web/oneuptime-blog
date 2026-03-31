# How to Configure K3s for Low-Resource Environments - Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Edge Computing, IoT, Raspberry Pi, Optimization, DevOps

Description: Learn how to tune and optimize K3s to run efficiently on resource-constrained hardware like Raspberry Pi, single-board computers, and low-memory servers.

## Introduction

K3s was designed to run on hardware as small as a Raspberry Pi, but even K3s needs careful tuning to run efficiently on very resource-constrained systems. With the right configuration, K3s can run stable Kubernetes workloads on systems with as little as 512MB RAM and a single CPU core. This guide covers comprehensive optimization techniques for low-resource K3s deployments.

## Minimum Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 1 core | 2 cores |
| RAM (Server) | 512MB | 1GB |
| RAM (Agent) | 256MB | 512MB |
| Storage | 4GB | 16GB |
| Architecture | ARM64/x86_64 | ARM64/x86_64 |

## Step 1: Disable Unnecessary K3s Components

The biggest wins come from disabling unused built-in components:

```yaml
# /etc/rancher/k3s/config.yaml

# Disable all non-essential components
disable:
  - traefik         # ~50MB RAM savings
  - metrics-server  # ~30MB RAM savings
  - coredns         # Only if you don't need DNS (very edge case)
  - local-storage   # If using external storage

# Note: coredns is usually needed for service discovery
# Only disable metrics-server if you don't need HPA or kubectl top
```

## Step 2: Optimize Kubelet Settings

Tune kubelet for resource-constrained environments:

```yaml
# /etc/rancher/k3s/config.yaml
kubelet-arg:
  # Limit number of pods (default 110 is too high for edge)
  - "max-pods=30"

  # Reserve resources for the OS and K3s system processes
  # Prevent pods from consuming all system resources
  - "system-reserved=cpu=100m,memory=256Mi,ephemeral-storage=1Gi"
  - "kube-reserved=cpu=100m,memory=256Mi,ephemeral-storage=512Mi"

  # Evict pods early to prevent OOM crashes
  - "eviction-hard=memory.available<100Mi,nodefs.available<5%,nodefs.inodesFree<5%"
  - "eviction-soft=memory.available<200Mi,nodefs.available<10%"
  - "eviction-soft-grace-period=memory.available=1m30s,nodefs.available=1m30s"

  # Reduce image GC thresholds to reclaim disk space
  - "image-gc-low-threshold=50"
  - "image-gc-high-threshold=70"

  # Less frequent node status updates to reduce CPU usage
  - "node-status-update-frequency=10s"

  # Reduce default CPU manager period
  - "cpu-manager-reconcile-period=10s"
```

## Step 3: Optimize kube-apiserver

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  # Reduce watch timeout to free connections faster
  - "default-watch-cache-size=100"

  # Limit request body size for resource protection
  - "max-requests-inflight=150"
  - "max-mutating-requests-inflight=50"

kube-controller-manager-arg:
  # Reduce concurrent syncs to lower CPU usage
  - "concurrent-deployment-syncs=2"
  - "concurrent-endpoint-syncs=2"
  - "concurrent-service-syncs=1"
  - "concurrent-namespace-syncs=5"
  - "concurrent-replicaset-syncs=5"
```

## Step 4: Optimize containerd

```toml
# /etc/rancher/k3s/config.toml (if using external containerd)
# Or configure via K3s's embedded containerd

# Reduce snapshot storage with overlayfs
[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "overlayfs"
  discard_unpacked_layers = true
```

## Step 5: Use SQLite Instead of etcd

For single-server deployments, K3s defaults to SQLite which is much lighter than etcd:

```bash
# Verify you're using SQLite (default for single-server)
ls /var/lib/rancher/k3s/server/db/
# Should see: state.db (SQLite) not etcd directory

# If you accidentally started with embedded etcd,
# reinstall as single-server for SQLite
```

## Step 6: Configure Swap (ARM/Low-RAM Devices)

For devices with very limited RAM:

```bash
# Create swap file to prevent OOM kills
# (Generally not recommended for Kubernetes, but useful for extreme edge)
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile swap swap defaults 0 0' >> /etc/fstab

# Reduce swap aggressiveness (Linux will prefer RAM over swap)
echo 'vm.swappiness = 10' >> /etc/sysctl.conf
echo 'vm.vfs_cache_pressure = 50' >> /etc/sysctl.conf
sysctl -p

# Note: K3s can work with swap, but Kubernetes normally requires
# swap to be disabled. Use with caution.
# To allow swap with K3s:
```

```yaml
# /etc/rancher/k3s/config.yaml
kubelet-arg:
  # Allow swap (required if swap is enabled)
  - "fail-swap-on=false"
```

## Step 7: Optimize Container Images for Edge

Use minimal base images in your workloads:

```dockerfile
# Use Alpine instead of Debian/Ubuntu
FROM alpine:3.19

# Or use distroless for even smaller images
FROM gcr.io/distroless/base-debian12

# Or scratch for Go binaries
FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

## Step 8: Configure Resource Limits on Workloads

Always set resource requests and limits:

```yaml
# resource-constrained-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-sensor-app
spec:
  replicas: 1  # Only 1 replica on small edge nodes
  template:
    spec:
      containers:
        - name: sensor-app
          image: myregistry/sensor:v1-alpine  # Use Alpine-based image
          resources:
            requests:
              cpu: 50m       # Very small CPU request
              memory: 64Mi   # Small memory request
            limits:
              cpu: 200m      # Cap at 0.2 CPU cores
              memory: 128Mi  # Hard memory limit
          # Use ConfigMap for config (not embedded)
          envFrom:
            - configMapRef:
                name: sensor-config
```

## Step 9: Use Local Storage Efficiently

```yaml
# local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-edge-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/edge-storage
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - edge-node-01
```

## Step 10: Monitor Resource Usage on Low-Resource Nodes

```bash
# Check current memory usage by K3s components
ps aux | grep -E "k3s|containerd|kubelet" | \
  awk '{print $11, $6/1024 "MB RSS"}'

# Monitor memory pressure
watch -n 5 'free -h && echo "---" && ps aux --sort=-%mem | head -15'

# Check K3s binary sizes
ls -lh /usr/local/bin/k3s

# Check containerd image storage
du -sh /var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter*

# Prune unused images
k3s crictl rmi --prune

# Check disk usage
df -h
du -sh /var/lib/rancher/k3s/
```

## Conclusion

K3s excels in low-resource environments when properly tuned. The most impactful optimizations are disabling unused built-in components (Traefik, metrics-server), setting appropriate kubelet resource reservations to prevent OS starvation, using minimal container images, and always specifying resource requests/limits on workloads. For very constrained devices like Raspberry Pi 3, a single-server K3s deployment with 2-3 small workloads is achievable and production-viable.
