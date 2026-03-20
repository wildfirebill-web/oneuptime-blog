# How to Configure RKE with Custom Docker Options

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE, Kubernetes, Rancher, Docker, Container Runtime, Configuration

Description: Learn how to configure custom Docker daemon options and per-service Docker settings in your RKE cluster.

## Introduction

RKE uses Docker as its container runtime. You can customize Docker daemon behavior globally across your cluster or on individual nodes. This is useful for configuring insecure registries, custom DNS, log drivers, storage drivers, and resource constraints for Kubernetes component containers.

## Configuring the Docker Daemon on Cluster Nodes

Before RKE can use custom Docker options, configure the Docker daemon on each node.

```bash
# Create or edit the Docker daemon configuration
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "insecure-registries": ["registry.example.com:5000"],
  "registry-mirrors": ["https://mirror.example.com"],
  "bip": "172.17.0.1/16",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
EOF

# Restart Docker to apply settings
sudo systemctl restart docker

# Verify the configuration
docker info
```

## Configuring Docker Options in cluster.yml

RKE allows you to pass Docker configuration options at the service level in `cluster.yml`.

### Configuring Docker for All Services

```yaml
# cluster.yml
# Docker image pull policy
system_images:
  kubernetes: rancher/hyperkube:v1.28.8-rancher1
  # Other system images...

# Configure docker volume mounts for services
services:
  kube-api:
    # Extra environment variables for the kube-apiserver container
    extra_env:
      - "GOGC=50"
    # Extra volume mounts (host:container)
    extra_binds:
      - "/etc/ssl/certs:/etc/ssl/certs:ro"
      - "/var/log/audit:/var/log/audit:rw"
    # Extra Docker labels
    extra_labels:
      - "environment=production"

  kubelet:
    extra_env:
      - "KUBELET_MAX_PODS=250"
    extra_binds:
      - "/sys/fs/cgroup:/sys/fs/cgroup:rw"
      - "/dev:/dev:rw"

  etcd:
    extra_binds:
      - "/var/lib/etcd:/var/lib/etcd:rw"
    # Docker resource limits for the etcd container
    image: rancher/mirrored-coreos-etcd:v3.5.9
```

### Configuring Private Registries

```yaml
# cluster.yml
private_registries:
  # Docker Hub mirror
  - url: registry.example.com
    user: admin
    password: "SuperSecretPassword"
    is_default: false

  # Default registry for all pulls
  - url: my-registry.internal:5000
    user: rke-user
    password: "AnotherPassword"
    is_default: true
```

### Configuring Per-Node Docker Settings

You can specify different Docker socket paths or other per-node options:

```yaml
# cluster.yml
nodes:
  - address: 192.168.1.101
    user: ubuntu
    role: [controlplane, etcd]
    # Use a non-default Docker socket
    docker_socket: /var/run/docker.sock
    # SSH connection options
    ssh_key_path: ~/.ssh/id_ed25519
    ssh_agent_auth: false

  - address: 192.168.1.102
    user: ubuntu
    role: [worker]
    docker_socket: /var/run/docker.sock
```

## Configuring Log Drivers per Service

Control how Kubernetes component logs are stored:

```yaml
# cluster.yml
services:
  kube-api:
    # Use systemd journal for API server logs
    extra_labels:
      - "logging=json"

  kubelet:
    extra_args:
      # Configure container log rotation via kubelet
      container-log-max-size: "50Mi"
      container-log-max-files: "5"
```

## Configuring Docker Storage for Large Clusters

For clusters with many images, tune Docker's storage backend:

```bash
# /etc/docker/daemon.json - Storage optimization
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=20G"
  ],
  "data-root": "/var/lib/docker",
  "exec-root": "/var/run/docker"
}
```

## Using Docker Behind a Proxy

For air-gapped or proxy environments:

```bash
# Configure Docker to use an HTTP proxy
sudo mkdir -p /etc/systemd/system/docker.service.d

sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null <<EOF
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.0.0/16,.example.com"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

Then configure the same proxy in the RKE engine config:

```yaml
# cluster.yml
# Pass proxy settings to Kubernetes components via extra_env
services:
  kube-controller:
    extra_env:
      - "HTTP_PROXY=http://proxy.example.com:3128"
      - "HTTPS_PROXY=http://proxy.example.com:3128"
      - "NO_PROXY=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
```

## Verifying Docker Configuration

```bash
# Check Docker daemon info
docker info

# Verify insecure registries
docker info | grep -A 5 "Insecure Registries"

# Verify log driver settings
docker info | grep "Logging Driver"

# Test pulling from a private registry
docker pull registry.example.com:5000/my-image:latest
```

## Docker Version Compatibility

Ensure Docker versions are compatible with your target Kubernetes version:

```bash
# Check Docker version on each node
for NODE in 192.168.1.101 192.168.1.102 192.168.1.103; do
    echo "Node $NODE: $(ssh ubuntu@$NODE docker version --format '{{.Server.Version}}')"
done

# Check RKE compatibility matrix
rke config --list-version --all | head -20
```

## Troubleshooting Docker Issues

### Container fails to start due to resource limits

```bash
# Check Docker events for errors
docker events --filter type=container --since 30m

# Check system resource limits
ulimit -a
cat /proc/sys/fs/inotify/max_user_watches
```

### Overlay2 storage issues

```bash
# Check for overlay2 errors
dmesg | grep overlay

# Verify the kernel supports overlay2
grep CONFIG_OVERLAY_FS /boot/config-$(uname -r)
```

## Conclusion

Configuring Docker options in RKE gives you fine-grained control over the container runtime behavior across your cluster. From setting up private registries and log rotation to configuring proxy settings and storage drivers, the combination of Docker daemon configuration and RKE's `cluster.yml` service options provides a comprehensive toolkit for production-ready deployments. Always configure these settings before running `rke up` to ensure a consistent cluster state.
