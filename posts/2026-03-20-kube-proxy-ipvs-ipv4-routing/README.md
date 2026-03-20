# How to Configure kube-proxy in IPVS Mode for IPv4 Service Routing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, kube-proxy, IPVS, IPv4, Service Routing, Performance

Description: Switch kube-proxy from iptables to IPVS mode for improved IPv4 service routing performance in large Kubernetes clusters with many services.

IPVS (IP Virtual Server) mode uses Linux's in-kernel load balancer instead of iptables chains. It provides O(1) lookup time (vs O(n) for iptables), supports more load balancing algorithms, and scales significantly better for large clusters.

## Prerequisites

```bash
# Load required kernel modules on all nodes
sudo modprobe ip_vs
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr
sudo modprobe ip_vs_sh
sudo modprobe nf_conntrack

# Make persistent
cat >> /etc/modules << 'EOF'
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

# Verify modules are loaded
lsmod | grep -e ip_vs -e nf_conntrack

# Install ipvsadm for inspection
sudo apt install ipvsadm -y
```

## Switching kube-proxy to IPVS Mode

```bash
# Edit the kube-proxy ConfigMap
kubectl edit configmap kube-proxy -n kube-system
```

Change the mode field:

```yaml
# In the kube-proxy ConfigMap
mode: "ipvs"
ipvs:
  # Load balancing algorithm (options: rr, lc, dh, sh, sed, nq)
  scheduler: "rr"
  # Sync period
  syncPeriod: 30s
  minSyncPeriod: 10s
  # Timeout for IPVS TCP connections
  tcpTimeout: 0s
  # UDP timeout
  udpTimeout: 0s
```

```bash
# Restart kube-proxy to apply the change
kubectl rollout restart daemonset/kube-proxy -n kube-system

# Verify the mode changed
kubectl logs -n kube-system $(kubectl get pods -n kube-system -l k8s-app=kube-proxy -o name | head -1) \
  | grep -i "mode\|ipvs"
# Expected: "Using ipvs Proxier"
```

## Verifying IPVS Rules

```bash
# View all IPVS virtual services (each Kubernetes service)
sudo ipvsadm -L -n

# Example output:
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port Forward Weight ActiveConn InActConn
# TCP  10.96.0.1:443 rr
#   -> 192.168.1.10:6443         Masq    1      0          0
# TCP  10.96.45.123:80 rr
#   -> 10.244.1.5:8080          Masq    1      0          0
#   -> 10.244.2.8:8080          Masq    1      0          0
```

## IPVS Load Balancing Algorithms

```bash
# Available schedulers:
# rr  - Round Robin (default)
# lc  - Least Connection
# dh  - Destination Hashing
# sh  - Source Hashing (session persistence)
# sed - Shortest Expected Delay
# nq  - Never Queue

# Change to Least Connection for better distribution under varying load
kubectl edit configmap kube-proxy -n kube-system
# Set: scheduler: "lc"
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

## Performance Comparison

IPVS scales much better than iptables:

| Metric | iptables (10K services) | IPVS (10K services) |
|---|---|---|
| Rule lookup time | O(n) linear | O(1) hash |
| CPU on sync | High | Low |
| Memory | Higher | Lower |
| Connection tracking | iptables conntrack | IPVS internal |

## Monitoring IPVS Connections

```bash
# View active connections
sudo ipvsadm -L -n --stats

# View connection rates per service
sudo ipvsadm -L -n --rate

# Check connection persistence entries
sudo ipvsadm -L -n --persistent-conn
```

For clusters with more than a few hundred services, IPVS mode is strongly recommended for its O(1) routing performance and lower CPU overhead.
