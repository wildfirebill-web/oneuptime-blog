# How to Configure K3s for Low-Resource Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Edge Computing, Low Resource, IoT, Embedded Systems, Kubernetes, SUSE Rancher

Description: Learn how to configure K3s for low-resource environments like Raspberry Pi, IoT gateways, and embedded systems by disabling unused components and tuning resource usage.

---

K3s was designed to run in resource-constrained environments. With the right configuration, it can run on devices with as little as 512 MB of RAM and 1 CPU core.

---

## Minimum Requirements

| Configuration | CPU | RAM | Storage |
|---|---|---|---|
| Single node (no workloads) | 1 core | 512 MB | 1 GB |
| Single node (with workloads) | 1 core | 1 GB | 5 GB |
| Recommended for production | 2 cores | 2 GB | 20 GB |

---

## Step 1: Disable Unnecessary Components

```bash
# Install K3s with all non-essential components disabled
curl -sfL https://get.k3s.io | sh -s - \
  --disable traefik \          # Remove if you don't need ingress
  --disable servicelb \        # Remove if you use MetalLB or NodePort only
  --disable local-storage \    # Remove if using external storage
  --disable metrics-server \   # Remove if not monitoring
  --disable coredns            # Only remove if using host DNS
```

---

## Step 2: Configure Resource Limits for K3s Components

```yaml
# /etc/rancher/k3s/config.yaml
kubelet-arg:
  - "max-pods=50"                          # Reduce from default 110
  - "eviction-hard=memory.available<100Mi" # Evict pods early to avoid OOM
  - "system-reserved=cpu=200m,memory=200Mi" # Reserve resources for OS
  - "kube-reserved=cpu=200m,memory=200Mi"   # Reserve resources for K3s
  - "image-gc-high-threshold=80"           # Start GC at 80% disk usage
  - "image-gc-low-threshold=70"            # GC until 70% disk usage

kube-controller-manager-arg:
  - "node-monitor-period=10s"              # Reduce monitoring frequency
  - "node-monitor-grace-period=40s"

kube-apiserver-arg:
  - "default-watch-cache-size=0"           # Disable watch cache to save memory
```

---

## Step 3: Use SQLite Instead of etcd

K3s defaults to SQLite for single-node deployments — this is far more memory-efficient than etcd:

```bash
# SQLite is used automatically for single-node clusters
# No extra configuration needed
# Verify:
ls /var/lib/rancher/k3s/server/db/state.db    # SQLite database file
```

For embedded HA with etcd, stick with 3 nodes minimum and adequate resources.

---

## Step 4: Reduce containerd Memory Usage

```toml
# /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  # Reduce max concurrent downloads
  [plugins."io.containerd.grpc.v1.cri".containerd]
    max_concurrent_downloads = 2         # Reduce from default 3

  # Disable unused runtimes
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

---

## Step 5: Set Pod Resource Limits

Configure LimitRange to prevent workloads from consuming all available resources:

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: default
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 128Mi
      defaultRequest:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: 500m
        memory: 256Mi
```

```bash
kubectl apply -f limitrange.yaml
```

---

## Step 6: Enable Swap (for very memory-constrained devices)

Kubernetes 1.28+ supports swap if the kubelet is configured to allow it:

```yaml
# /etc/rancher/k3s/config.yaml
kubelet-arg:
  - "feature-gates=NodeSwap=true"
  - "fail-swap-on=false"
  - "memory-swap=0"              # Unlimited swap (use with caution)
```

```bash
# Enable swap on the host
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

---

## Step 7: Monitor Resource Usage

```bash
# Check K3s process memory usage
ps aux | grep k3s

# Check node resource usage
kubectl top nodes

# Check pod resource usage
kubectl top pods -A

# Monitor with watch
watch -n 5 kubectl top nodes
```

---

## Best Practices

- Always disable `traefik` and `servicelb` on resource-constrained nodes unless you specifically need them — they consume significant memory.
- Set `max-pods` to match the expected workload density rather than using the default 110 — this prevents K3s from accepting more pods than the node can handle.
- For IoT devices with intermittent power, enable K3s auto-restart: the systemd service is configured to restart automatically after crashes by default.
