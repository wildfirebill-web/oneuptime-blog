# How to Configure RKE2 SELinux Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, SELinux, Security, Linux, Rancher

Description: Learn how to configure and enable SELinux support in RKE2 for enhanced mandatory access control security on RHEL, CentOS, and Rocky Linux nodes.

SELinux (Security-Enhanced Linux) is a mandatory access control (MAC) security mechanism built into the Linux kernel. When properly configured with RKE2, SELinux adds an additional layer of security that restricts what processes can access, even if they gain elevated privileges. This guide covers enabling and configuring SELinux support in RKE2.

## Prerequisites

- RHEL, CentOS, Rocky Linux, or Fedora with SELinux available
- RKE2 v1.21+
- Understanding of SELinux concepts (contexts, policies, booleans)

## Understanding SELinux with Kubernetes

SELinux can conflict with Kubernetes container runtime operations if not properly configured. RKE2 provides an SELinux policy package (`rke2-selinux`) that defines the necessary SELinux contexts for RKE2 components.

## Step 1: Check Current SELinux Status

```bash
# Check SELinux status

sestatus

# Check current enforcement mode
getenforce

# Available modes:
# Enforcing - SELinux policy is enforced
# Permissive - SELinux logs but doesn't enforce
# Disabled - SELinux is disabled

# For production: use Enforcing
# For testing/debugging: use Permissive
```

## Step 2: Install RKE2 SELinux Policy

```bash
# For RHEL/CentOS/Rocky Linux, install the RKE2 SELinux policy
# First, enable the container-selinux policy
sudo dnf install -y container-selinux

# Install RKE2-specific SELinux policy
# Option 1: From a repository
sudo dnf install -y https://rpm.rancher.io/rke2/latest/centos/8/noarch/rke2-selinux-0.17-1.el8.noarch.rpm

# Option 2: Install during RKE2 installation
curl -sfL https://get.rke2.io | \
  INSTALL_RKE2_SELINUX=true sudo sh -

# Verify the policy is installed
rpm -q rke2-selinux
```

## Step 3: Configure SELinux Mode

```bash
# Set SELinux to enforcing mode
sudo setenforce 1

# Make it permanent
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config

# Verify
cat /etc/selinux/config | grep "^SELINUX="
```

## Step 4: Configure RKE2 to Work with SELinux

```yaml
# /etc/rancher/rke2/config.yaml - SELinux configuration
# Enable SELinux in the container runtime
selinux: true

# Additional kubelet args for SELinux
kubelet-arg:
  # Enable SELinux context labeling
  - "seccomp-default=true"

# The selinux: true setting tells containerd to:
# 1. Apply SELinux labels to containers
# 2. Use the container_t policy context for containers
# 3. Respect SELinux labels on volume mounts
```

## Step 5: Verify SELinux Labels on RKE2 Processes

```bash
# Start RKE2
sudo systemctl start rke2-server

# Check SELinux labels on RKE2 processes
ps -eZ | grep -E "rke2|containerd|kubelet"

# Check SELinux context of container processes
# Example: etcd, kube-apiserver should have container_t context
ps -eZ | grep -E "etcd|kube-apiserver"

# Check SELinux labels on RKE2 files
ls -laZ /var/lib/rancher/rke2/
ls -laZ /etc/rancher/rke2/
```

## Step 6: Troubleshoot SELinux Denials

When SELinux denials occur, use these tools to diagnose:

```bash
# View recent SELinux denials
sudo ausearch -m avc --start recent

# Format denials for readability
sudo ausearch -m avc --start recent | audit2why

# Get suggested policy adjustments
sudo ausearch -m avc --start recent | audit2allow -M mypolicy

# View SELinux audit log
sudo cat /var/log/audit/audit.log | grep denied | tail -20

# Check if RKE2 is generating denials
sudo ausearch -m avc -c rke2 --start recent
sudo ausearch -m avc -c containerd --start recent
sudo ausearch -m avc -c kubelet --start recent
```

## Step 7: Configure SELinux Boolean Settings

Some RKE2 features require specific SELinux booleans:

```bash
# Check current SELinux booleans relevant to containers
getsebool -a | grep -E "container|docker|virt"

# Enable required booleans for RKE2
# Allow containers to manage network configuration
sudo setsebool -P container_manage_cgroup on

# Allow containers to use NFS volumes
sudo setsebool -P container_use_nfs on

# Allow containers to read/write to /var/lib/kubelet
sudo setsebool -P container_connect_any on

# Verify boolean settings
getsebool container_manage_cgroup
getsebool container_use_nfs
```

## Step 8: SELinux Contexts for Persistent Volumes

When using persistent volumes, SELinux contexts must be correct:

```bash
# Check the SELinux context of a volume directory
ls -laZ /data/

# Set the correct context for Kubernetes data directories
sudo semanage fcontext -a \
  -t container_file_t "/data(/.*)?"

sudo restorecon -Rv /data/

# Verify the context was applied
ls -laZ /data/
# Should show container_file_t
```

```yaml
# pod-with-selinux.yaml - Configure SELinux context in pod spec
apiVersion: v1
kind: Pod
metadata:
  name: selinux-pod
spec:
  securityContext:
    seLinuxOptions:
      # Use a specific SELinux context for all containers
      level: "s0:c123,c456"
      # Or use a specific type
      type: "container_t"
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      seLinuxOptions:
        # Container-specific override
        level: "s0:c789,c012"
```

## Conclusion

Configuring SELinux with RKE2 provides an additional layer of mandatory access control that protects against container escapes and privilege escalation. The RKE2 SELinux policy package (`rke2-selinux`) simplifies the process by pre-defining the necessary policies for RKE2 components. When deploying in RHEL, CentOS, or Rocky Linux environments, enabling SELinux enforcement from the start is strongly recommended for production clusters, especially those requiring CIS benchmark or STIG compliance.
