# How to Configure K3s to Use containerd

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kubernetes, Rancher, Containerd, Container Runtime

Description: Learn how to configure and customize the containerd runtime bundled with K3s, including snapshotter settings and containerd configuration overrides.

## Introduction

K3s ships with an embedded containerd runtime, which it manages internally. Unlike Docker-based runtimes, containerd is a lightweight, industry-standard container runtime that provides excellent performance and is the default for Kubernetes. This guide covers how to configure the embedded containerd, use external containerd installations, and tune containerd for specific workloads.

## K3s and containerd

K3s bundles its own containerd binary at `/var/lib/rancher/k3s/data/<version>/bin/containerd`. This embedded containerd is configured and managed by K3s automatically.

Key paths:
- containerd socket: `/run/k3s/containerd/containerd.sock`
- containerd config: `/var/lib/rancher/k3s/agent/etc/containerd/config.toml`
- containerd data: `/var/lib/rancher/k3s/agent/containerd/`

## Viewing the Current containerd Configuration

```bash
# View the generated containerd config

sudo cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml

# Use the bundled ctr tool to interact with containerd
sudo k3s ctr version
sudo k3s ctr images list
sudo k3s ctr containers list
sudo k3s ctr namespaces list
```

## Customizing containerd with a Config Template

K3s generates the containerd config from a template. You can override it by providing your own template:

```bash
# Create the custom config template directory
sudo mkdir -p /var/lib/rancher/k3s/agent/etc/containerd/

# Create a config template (K3s will use this instead of generating its own)
sudo tee /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl > /dev/null <<'TOML'
version = 2

[plugins."io.containerd.internal.v1.opt"]
  path = "{{ .NodeConfig.Containerd.Opt }}"

[plugins."io.containerd.grpc.v1.cri"]
  stream_server_address = "127.0.0.1"
  stream_server_port = "10010"
  enable_selinux = false
  enable_unprivileged_ports = false
  enable_unprivileged_icmp = false

{{- if .IsRunningInUserNS }}
  disable_cgroup = true
  disable_apparmor = true
  restrict_oom_score_adj = true
{{ end -}}

  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "{{ .NodeConfig.Containerd.Snapshotter }}"
    disable_snapshot_annotations = true
{{ if .NodeConfig.AgentConfig.Snapshotter }}
    snapshotter = "{{ .NodeConfig.AgentConfig.Snapshotter }}"
{{ end }}

    default_runtime_name = "runc"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = {{ .NodeConfig.AgentConfig.Systemd }}

  [plugins."io.containerd.grpc.v1.cri".registry]
    config_path = "/var/lib/rancher/k3s/agent/etc/containerd/certs.d"
TOML
```

## Configuring the Snapshotter

The snapshotter determines how container layers are stored. Options include `overlayfs` (default), `fuse-overlayfs`, `btrfs`, and `zfs`.

```bash
# Configure K3s to use a specific snapshotter
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
# Use fuse-overlayfs for unprivileged container support
snapshotter: fuse-overlayfs
EOF
```

For NVMe-optimized installations, use the btrfs snapshotter:

```bash
# Format the data directory partition as btrfs
sudo mkfs.btrfs /dev/nvme0n1p1
sudo mount /dev/nvme0n1p1 /var/lib/rancher

# Configure K3s to use btrfs snapshotter
sudo tee -a /etc/rancher/k3s/config.yaml > /dev/null <<EOF
snapshotter: btrfs
EOF
```

## Configuring containerd Runtime Classes

Add additional container runtimes (e.g., gVisor, Kata Containers) via the config template:

```bash
# Add gVisor (runsc) as an alternative runtime
sudo tee /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl > /dev/null <<'TOML'
version = 2

[plugins."io.containerd.grpc.v1.cri".containerd]
  default_runtime_name = "runc"

  # Standard runc runtime
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true

  # gVisor runtime (must have runsc installed at /usr/local/bin/runsc)
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
    runtime_type = "io.containerd.runsc.v1"
TOML

# Restart K3s to apply
sudo systemctl restart k3s
```

Create a RuntimeClass for gVisor:

```yaml
# gvisor-runtimeclass.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

```bash
kubectl apply -f gvisor-runtimeclass.yaml

# Use gVisor for a pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gvisor-test
spec:
  runtimeClassName: gvisor
  containers:
    - name: test
      image: alpine
      command: ["sleep", "3600"]
EOF
```

## Configuring containerd Image Garbage Collection

```bash
# Configure GC via containerd config template
sudo tee -a /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl > /dev/null <<'TOML'

[plugins."io.containerd.gc.v1.scheduler"]
  # Pause between GC cycles (default: 100ms)
  pause_threshold = 0.02
  # Minimum time before an unreferenced content is garbage collected
  deletion_threshold = 0
  # Minimum content a schedule must be needed
  mutation_threshold = 100
  # Schedule GC every n seconds
  schedule_delay = "0s"
  # Start GC at this percentage of content store usage
  startup_delay = "100ms"
TOML
```

## Using the Bundled crictl

K3s installs `crictl` for containerd debugging:

```bash
# Configure crictl to use K3s's containerd socket
export CONTAINER_RUNTIME_ENDPOINT="unix:///run/k3s/containerd/containerd.sock"

# Or create a permanent config
sudo tee /etc/crictl.yaml > /dev/null <<EOF
runtime-endpoint: unix:///run/k3s/containerd/containerd.sock
image-endpoint: unix:///run/k3s/containerd/containerd.sock
timeout: 10
debug: false
EOF

# Now use crictl commands
sudo crictl ps          # List running containers
sudo crictl images      # List images
sudo crictl pods        # List pods
sudo crictl logs <container-id>  # View container logs
```

## Pre-Loading Images into containerd

```bash
# Import a saved image directly into K3s's containerd
sudo k3s ctr images import /path/to/image.tar

# Or use crictl
sudo crictl pull my-image:latest

# Verify images
sudo k3s ctr images list | grep my-image
```

## Monitoring containerd Performance

```bash
# Check containerd resource usage
sudo systemctl status k3s | grep -A 5 "CGroup"

# Monitor containerd metrics (if enabled)
curl -s http://localhost:1338/metrics | grep containerd

# Check containerd events
sudo k3s ctr events
```

## Conclusion

K3s's embedded containerd provides a robust container runtime with sensible defaults. Most users don't need to modify the containerd configuration, but when customization is needed - such as adding GPU or sandbox runtimes, tuning the snapshotter, or configuring custom GC settings - the `config.toml.tmpl` template mechanism provides a clean way to override defaults. The bundled `ctr` and `crictl` tools give you direct access to containerd for debugging and image management.
