# How to Install K3s on Alpine Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Alpine Linux, Lightweight, Edge Computing

Description: Step-by-step guide to installing K3s on Alpine Linux, addressing its unique musl libc and OpenRC init system requirements.

## Introduction

Alpine Linux is a security-focused, minimal Linux distribution that uses musl libc and BusyBox instead of glibc and GNU tools. Its small footprint makes it attractive for container hosts and edge deployments. Installing K3s on Alpine requires a few extra steps due to its non-standard init system (OpenRC instead of systemd) and musl libc compatibility requirements.

## Alpine Version Requirements

- Alpine Linux 3.14 or newer is recommended
- K3s v1.21+ for best Alpine compatibility
- Both x86_64 and ARM64 are supported

## Step 1: Install Alpine Linux

If installing from scratch, use the Alpine Extended ISO (includes networking tools):

```bash
# After installation, update and install required packages
apk update && apk upgrade

# Install required packages
apk add --no-cache \
    curl \
    bash \
    coreutils \
    findutils \
    util-linux \
    mount \
    blkid \
    nfs-utils \
    iptables \
    ip6tables \
    cni-plugins

# Verify the architecture
uname -m
```

## Step 2: Enable Required Kernel Modules and cgroups

Alpine uses OpenRC and may not have cgroup v2 enabled by default:

```bash
# Enable required kernel modules
cat >> /etc/modules <<EOF
br_netfilter
overlay
ip_tables
ip6_tables
nf_conntrack
nf_conntrack_netlink
EOF

# Load them immediately
modprobe br_netfilter overlay ip_tables ip6_tables nf_conntrack

# Enable IPv4 forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
sysctl -p

# Enable cgroup memory (add to kernel boot parameters)
# Edit /etc/default/grub or the appropriate bootloader config
# For syslinux/extlinux:
sed -i 's/^APPEND.*/& cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1/' /boot/extlinux.conf

# For GRUB:
# sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1"/' /etc/default/grub
# grub-mkconfig -o /boot/grub/grub.cfg
```

```bash
# Add cgroup v1 mount if not present
mount -t cgroup2 none /sys/fs/cgroup 2>/dev/null || true

# Or add to /etc/fstab for persistence
echo "cgroup /sys/fs/cgroup cgroup defaults 0 0" >> /etc/fstab
```

## Step 3: Disable Swap

```bash
# Check and disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Verify
free -h
```

## Step 4: Install Required Glibc Compatibility

K3s's bundled components may require glibc compatibility:

```bash
# Install compatibility packages
apk add --no-cache \
    libc6-compat \
    libgcc \
    libstdc++

# Verify compat libraries
ls /lib/libgcc_s.so.1
```

## Step 5: Install K3s

```bash
# Create configuration directory
mkdir -p /etc/rancher/k3s

# Create K3s configuration
cat > /etc/rancher/k3s/config.yaml <<EOF
token: "AlpineK3sToken"
tls-san:
  - $(hostname -I | awk '{print $1}')
  - $(hostname)
disable:
  - servicelb  # Use iptables-based LB for Alpine compatibility
kubelet-arg:
  - "max-pods=110"
  - "resolv-conf=/etc/resolv.conf"
EOF

# Install K3s (skip systemd service setup since Alpine uses OpenRC)
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_SKIP_ENABLE=true \
    sh -
```

## Step 6: Configure OpenRC Service

Since Alpine uses OpenRC instead of systemd, create an OpenRC init script:

```bash
# Create the OpenRC init script for K3s
cat > /etc/init.d/k3s <<'EOF'
#!/sbin/openrc-run

name="k3s"
description="Lightweight Kubernetes"
command="/usr/local/bin/k3s"
command_args="server"
command_background=true
pidfile="/var/run/k3s.pid"
output_log="/var/log/k3s.log"
error_log="/var/log/k3s.log"

depend() {
    need net
    after firewall
}

start_pre() {
    # Ensure required modules are loaded
    modprobe br_netfilter 2>/dev/null || true
    modprobe overlay 2>/dev/null || true
    sysctl -w net.ipv4.ip_forward=1 >/dev/null 2>&1 || true
}
EOF

chmod +x /etc/init.d/k3s

# Enable and start the service
rc-update add k3s default
rc-service k3s start

# Check status
rc-service k3s status
```

## Step 7: Configure kubectl

```bash
# Set up kubeconfig
mkdir -p ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Test access (K3s installs kubectl)
k3s kubectl get nodes
# Or
kubectl get nodes
```

## Step 8: Configure an Agent Node on Alpine

For agent nodes, use a similar OpenRC setup:

```bash
# On the agent node
cat > /etc/rancher/k3s/config.yaml <<EOF
server: "https://SERVER_IP:6443"
token: "AlpineK3sToken"
EOF

# Install K3s as agent
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="agent" \
    INSTALL_K3S_SKIP_ENABLE=true \
    sh -

# Create OpenRC init script for the agent
cat > /etc/init.d/k3s-agent <<'EOF'
#!/sbin/openrc-run

name="k3s-agent"
description="Lightweight Kubernetes Agent"
command="/usr/local/bin/k3s"
command_args="agent"
command_background=true
pidfile="/var/run/k3s-agent.pid"
output_log="/var/log/k3s-agent.log"
error_log="/var/log/k3s-agent.log"

depend() {
    need net
}
EOF

chmod +x /etc/init.d/k3s-agent
rc-update add k3s-agent default
rc-service k3s-agent start
```

## Troubleshooting Alpine-Specific Issues

### CNI Plugins Not Found

```bash
# Install CNI plugins manually if not included
apk add --no-cache cni-plugins

# Copy to the K3s CNI directory
mkdir -p /opt/cni/bin
cp /usr/lib/cni/* /opt/cni/bin/
```

### iptables Not Working

```bash
# Alpine may need legacy iptables
apk add --no-cache iptables-legacy

# Set iptables to use legacy mode
ln -sf /sbin/iptables-legacy /sbin/iptables
ln -sf /sbin/ip6tables-legacy /sbin/ip6tables
```

### DNS Resolution Failing in Pods

```bash
# Check the resolv.conf K3s is using
k3s kubectl exec -it <pod> -- cat /etc/resolv.conf

# Ensure the host resolv.conf is valid
cat /etc/resolv.conf

# Configure K3s to use a specific resolv.conf
# Add to /etc/rancher/k3s/config.yaml:
# kubelet-arg:
#   - "resolv-conf=/etc/resolv.conf"
```

## Conclusion

K3s works on Alpine Linux but requires extra attention to cgroup configuration, OpenRC service management, and glibc compatibility. Alpine's minimal footprint makes it an efficient K3s host when properly configured. The main challenges are the OpenRC init system (no systemd) and ensuring cgroup memory is enabled at boot. Once these are addressed, K3s runs well and benefits from Alpine's security-hardened, minimal base.
