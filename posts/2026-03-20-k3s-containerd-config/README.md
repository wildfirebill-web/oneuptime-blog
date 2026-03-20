# How to Configure K3s to Use containerd - Config

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Containerd, Container Runtime, Kubernetes, Configuration, SUSE Rancher

Description: Learn how to configure K3s with containerd including private registry authentication, mirror configuration, runtime settings, and debugging containerd issues.

---

K3s uses containerd as its default container runtime. Understanding how to configure containerd directly allows you to manage private registries, mirrors, and runtime settings without modifying K3s itself.

---

## containerd Configuration Location

K3s manages its own containerd instance, separate from any system-wide containerd:

```text
/var/lib/rancher/k3s/agent/etc/containerd/config.toml       # Active config (managed by K3s)
/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl  # Template (edit this one)
```

To customize containerd, edit the `.tmpl` file - K3s uses it to regenerate `config.toml` on startup.

---

## Step 1: Configure Private Registry Authentication

```yaml
# /etc/rancher/k3s/registries.yaml

mirrors:
  "registry.example.com":
    endpoint:
      - "https://registry.example.com"

configs:
  "registry.example.com":
    auth:
      username: myuser
      password: mypassword
    tls:
      insecure_skip_verify: false      # Set true only for self-signed certs in dev
```

Restart K3s to apply:

```bash
systemctl restart k3s
```

---

## Step 2: Configure Registry Mirrors

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  "docker.io":
    endpoint:
      - "https://mirror.example.com"
      - "https://registry-1.docker.io"    # Fallback to Docker Hub

  "ghcr.io":
    endpoint:
      - "https://ghcr-mirror.example.com"
      - "https://ghcr.io"
```

---

## Step 3: Customize the containerd Config Template

For advanced settings, create a containerd config template:

```toml
# /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "rancher/mirrored-pause:3.6"

  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"           # Default snapshotter
    default_runtime_name = "runc"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true            # Required for systemd cgroup driver

  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
        endpoint = ["https://mirror.example.com"]
```

---

## Step 4: Use nerdctl to Interact with containerd

K3s bundles `ctr` and the K3s CLI can also interact with containerd:

```bash
# List running containers (K3s uses a specific namespace)
k3s ctr containers ls

# List images
k3s ctr images ls

# Pull an image manually into K3s containerd
k3s ctr images pull docker.io/library/nginx:1.24

# Check containerd snapshotter
k3s ctr snapshots ls

# View containerd info
k3s ctr info
```

---

## Step 5: Debug containerd Issues

```bash
# Check containerd socket
ls -la /run/k3s/containerd/containerd.sock

# View containerd logs (K3s embeds containerd, logs go to journald)
journalctl -u k3s | grep containerd

# Check the active containerd config
cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml

# Check if a specific image was pulled
k3s ctr images ls | grep nginx

# Check if a container failed to start
k3s ctr tasks ls
```

---

## Step 6: Configure containerd for GPU Support

For GPU workloads, add the NVIDIA runtime to containerd:

```toml
# Add to /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
```

Then in your Pod spec, reference the runtime class:

```yaml
spec:
  runtimeClassName: nvidia
```

---

## Best Practices

- Always use `/etc/rancher/k3s/registries.yaml` for registry configuration instead of editing the containerd config directly - K3s reads registries.yaml and manages the containerd config automatically.
- After any containerd configuration change, verify by pulling a test image: `k3s ctr images pull docker.io/library/hello-world:latest`.
- Enable `SystemdCgroup = true` in the containerd runc options when running on systemd-based Linux distributions for correct cgroup management.
