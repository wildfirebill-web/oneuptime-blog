# How to Troubleshoot RKE2 Installation Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Troubleshooting, Installation, Debugging, Rancher

Description: A comprehensive troubleshooting guide for diagnosing and resolving common RKE2 installation and startup failures.

RKE2 installations can fail for various reasons including network issues, system configuration problems, port conflicts, and resource constraints. This guide provides a systematic approach to diagnosing and resolving the most common RKE2 installation failures.

## Prerequisites

- A Linux system where RKE2 installation is failing
- Root or sudo access
- Basic Linux troubleshooting skills

## Step 1: Check Basic System Requirements

Before diving into logs, verify the system meets basic requirements:

```bash
# Check OS and kernel version
cat /etc/os-release
uname -r

# Check available resources
echo "=== CPU ==="
nproc
cat /proc/cpuinfo | grep "model name" | head -1

echo "=== Memory ==="
free -h
# Minimum: 2GB RAM for server, 1GB for agent

echo "=== Disk Space ==="
df -h
# Need at least 10GB free on / or /var

echo "=== Swap ==="
swapon --summary
# Swap must be disabled for Kubernetes

echo "=== Network ==="
hostname -I
ip link show
```

## Step 2: Check RKE2 Service Status

```bash
# Check if RKE2 is running
sudo systemctl status rke2-server
# or for agent nodes:
sudo systemctl status rke2-agent

# Get the last 50 lines of logs
sudo journalctl -u rke2-server -n 50 --no-pager
# or
sudo journalctl -u rke2-agent -n 50 --no-pager

# Follow logs in real-time
sudo journalctl -u rke2-server -f

# Check for error keywords
sudo journalctl -u rke2-server | grep -iE "error|fail|fatal|panic"
```

## Step 3: Diagnose Common Startup Failures

### Failure: "address already in use"

```bash
# Check if required ports are already in use
for port in 6443 9345 2379 2380 10250 10257 10259 8472; do
  if sudo ss -tlnp | grep ":$port"; then
    echo "PORT $port IS IN USE"
  fi
done

# Find what's using a specific port
sudo ss -tlnp | grep :6443
sudo lsof -i :6443

# Kill the conflicting process if safe to do so
# sudo kill -9 <PID>
```

### Failure: "swap is enabled"

```bash
# Check swap status
swapon --summary

# Disable swap immediately
sudo swapoff -a

# Make permanent by commenting out swap in /etc/fstab
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify
free -h
swapon --summary  # Should show no swap
```

### Failure: "br_netfilter module not loaded"

```bash
# Load the required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Make permanent
cat <<EOF | sudo tee /etc/modules-load.d/rke2.conf
overlay
br_netfilter
EOF

# Configure required sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/99-rke2.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Verify
lsmod | grep -E "overlay|br_netfilter"
sysctl net.bridge.bridge-nf-call-iptables
```

### Failure: "certificate verify failed" or "x509"

```bash
# Check if the server URL is correct in agent config
cat /etc/rancher/rke2/config.yaml

# Verify the server is reachable
SERVER_IP="<your-server-ip>"
curl -k https://$SERVER_IP:9345

# Check if TLS SANs cover all needed addresses
# On server node:
openssl x509 -in /var/lib/rancher/rke2/server/tls/serving-kube-apiserver.crt \
  -noout -ext subjectAltName

# If the IP/hostname is missing, add it to tls-san in config.yaml
# and restart RKE2
```

### Failure: "failed to get node" or node not joining

```bash
# Check the token on the agent node
cat /etc/rancher/rke2/config.yaml | grep token

# Verify the token matches the server
# On server node:
sudo cat /var/lib/rancher/rke2/server/node-token

# If token mismatch, update the agent config
sudo vi /etc/rancher/rke2/config.yaml
# Update the token value

# Restart agent
sudo systemctl restart rke2-agent
```

## Step 4: Diagnose Network Issues

```bash
# Test connectivity between nodes
SERVER_IP="10.0.0.10"
AGENT_IP="10.0.0.20"

# From agent to server ports
echo "=== Testing server connectivity ==="
nc -zv $SERVER_IP 9345  # Registration port
nc -zv $SERVER_IP 6443  # API server port

# Test DNS resolution
nslookup $SERVER_IP

# Check firewall rules
sudo iptables -L -n | head -30
# or for firewalld:
sudo firewall-cmd --list-all

# Check if CNI is creating interfaces
ip link show | grep -E "cni|flannel|calico|cilium"
ip addr show | grep "10.42\|10.43"
```

## Step 5: Diagnose containerd Issues

```bash
# Check containerd status
sudo systemctl status containerd
sudo journalctl -u containerd -n 30 --no-pager

# RKE2 uses its own embedded containerd
# Check RKE2's containerd socket
ls -la /run/k3s/containerd/containerd.sock

# Check containerd images
sudo /var/lib/rancher/rke2/bin/crictl \
  --runtime-endpoint unix:///run/k3s/containerd/containerd.sock \
  images

# Check running containers
sudo /var/lib/rancher/rke2/bin/crictl \
  --runtime-endpoint unix:///run/k3s/containerd/containerd.sock \
  ps -a

# Check container logs for failing containers
sudo /var/lib/rancher/rke2/bin/crictl \
  --runtime-endpoint unix:///run/k3s/containerd/containerd.sock \
  logs <container-id>
```

## Step 6: Diagnose etcd Issues

```bash
# Check if etcd is running
sudo ps aux | grep etcd

# Check etcd health
ETCD_DIR="/var/lib/rancher/rke2/server/tls/etcd"
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=$ETCD_DIR/server-ca.crt \
  --cert=$ETCD_DIR/client.crt \
  --key=$ETCD_DIR/client.key \
  endpoint health

# Check etcd logs
sudo journalctl -u rke2-server | grep etcd | tail -20

# Check etcd data directory permissions
ls -la /var/lib/rancher/rke2/server/db/etcd/
# Should be owned by root with restricted permissions
```

## Step 7: Clean Up and Retry Failed Installation

If all else fails, perform a clean reinstall:

```bash
# WARNING: This will DESTROY all cluster data
# Only use on a fresh install that failed, not on a running cluster

# Run the RKE2 uninstall script
sudo /usr/local/bin/rke2-uninstall.sh
# or for agent nodes:
sudo /usr/local/bin/rke2-agent-uninstall.sh

# Remove all RKE2 data
sudo rm -rf /var/lib/rancher/rke2
sudo rm -rf /etc/rancher/rke2
sudo rm -rf /run/k3s
sudo rm -rf /run/flannel

# Clean up network interfaces
sudo ip link delete flannel.1 2>/dev/null
sudo ip link delete cni0 2>/dev/null

# Reinstall RKE2
curl -sfL https://get.rke2.io | sudo sh -

# Recreate configuration
sudo mkdir -p /etc/rancher/rke2/
cat <<EOF | sudo tee /etc/rancher/rke2/config.yaml
# Your configuration here
tls-san:
  - "$(hostname -I | awk '{print $1}')"
write-kubeconfig-mode: "0644"
EOF

# Start RKE2
sudo systemctl enable rke2-server
sudo systemctl start rke2-server
sudo journalctl -u rke2-server -f
```

## Common Error Messages Reference

| Error Message | Likely Cause | Solution |
|---|---|---|
| `bind: address already in use` | Port conflict | Kill conflicting process |
| `failed to find system:node ClusterRoleBinding` | Bootstrap issue | Clean reinstall |
| `context deadline exceeded` | Network timeout | Check firewall rules |
| `x509: certificate signed by unknown authority` | CA cert mismatch | Regenerate certs |
| `etcdserver: no leader` | etcd cluster issue | Check etcd quorum |
| `Failed to load kubelet config file` | Bad config | Check config syntax |

## Conclusion

Troubleshooting RKE2 installation failures requires a systematic approach starting from basic system requirements and working through to component-specific diagnostics. The most common causes of installation failures are swap not disabled, required ports being in use, kernel modules not loaded, and token mismatches between server and agent nodes. Always check `journalctl -u rke2-server -f` first — the error messages are usually descriptive and point directly to the root cause.
