# How to Avoid IPv4 Fragmentation in Docker and Container Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, MTU, Fragmentation, Containers, Networking, Linux

Description: Configure Docker network MTU settings to prevent packet fragmentation in containerized environments, including bridge networks, overlay networks, and Kubernetes.

## Introduction

Docker containers can experience packet fragmentation when their network MTU differs from the host's effective MTU. This happens because Docker's default bridge network uses MTU 1500, but if the host is in a cloud environment with a lower effective MTU (AWS VPC: 9001, Azure: 1500, some VPN environments: 1400-1450), packets get fragmented. In overlay networks (Swarm, Kubernetes), the encapsulation overhead further reduces the effective MTU.

## Diagnose MTU Issue in Docker

```bash
# Check host interface MTU:

ip link show eth0 | grep mtu

# Check Docker bridge MTU:
ip link show docker0 | grep mtu
# Default: mtu 1500

# Check inside a running container:
docker run --rm alpine ip link show eth0
# Shows container's virtual interface MTU

# Test fragmentation from inside container:
docker run --rm alpine ping -M do -s 1472 -c 3 8.8.8.8
# If this fails but host-level ping works: container MTU > effective path MTU
```

## Set Docker Bridge MTU

```bash
# Method 1: Docker daemon configuration:
cat > /etc/docker/daemon.json << 'EOF'
{
  "mtu": 1450
}
EOF
systemctl restart docker

# Verify new default MTU:
docker network inspect bridge | grep Mtu
# Should show: "com.docker.network.driver.mtu": "1450"

# New containers use this MTU automatically
# Existing containers need to be recreated

# Method 2: Set MTU when creating a new network:
docker network create --opt com.docker.network.driver.mtu=1450 mynetwork

# Verify:
docker network inspect mynetwork | grep mtu
```

## Calculate Correct MTU

```bash
# MTU for containers = Host Path MTU - Encapsulation Overhead

# Standard Ethernet (no overhead):
# Container MTU = 1500 (same as host)

# AWS VPC (MTU 9001 but effective for EC2 is 1500 unless jumbo enabled):
# Container MTU = 1500

# VXLAN overlay (Swarm/Kubernetes with Flannel):
# VXLAN overhead: 50 bytes (20 IP + 8 UDP + 8 VXLAN + 14 inner Ethernet)
# Container MTU = host_MTU - 50 = 1500 - 50 = 1450

# WireGuard on host:
# WireGuard overhead: ~80 bytes
# Container MTU = wg0_MTU - 50 (for VXLAN) = 1420 - 50 = 1370

python3 -c "
host_mtu = 1500
vxlan_overhead = 50   # for overlay networks
container_mtu = host_mtu - vxlan_overhead
print(f'Recommended container MTU: {container_mtu}')
"
```

## Kubernetes MTU Configuration

```yaml
# Kubernetes MTU depends on the CNI plugin:

# Flannel VXLAN (edit configmap):
# kubectl -n kube-system edit configmap kube-flannel-cfg
# In net-conf.json:
# {
#   "Network": "10.244.0.0/16",
#   "Backend": {
#     "Type": "vxlan",
#     "MTU": 1450    ← Add this
#   }
# }

# Calico:
# kubectl -n kube-system set env daemonset/calico-node FELIX_IPINIPMTU=1480
# Or for VXLAN:
# kubectl -n kube-system set env daemonset/calico-node FELIX_VXLANMTU=1450

# Weave Net:
# Set env variable WEAVE_MTU=1376 in weave DaemonSet
```

```bash
# Check current pod MTU in Kubernetes:
kubectl run mtu-test --image=alpine --rm -it --restart=Never -- \
  ip link show eth0

# Test from pod:
kubectl run mtu-test --image=alpine --rm -it --restart=Never -- \
  ping -M do -s 1422 -c 3 8.8.8.8
# Adjust size based on actual overlay overhead
```

## Docker Compose MTU

```yaml
# docker-compose.yml - specify network MTU:
version: '3'
services:
  app:
    image: myapp
    networks:
      - internal

  db:
    image: postgres
    networks:
      - internal

networks:
  internal:
    driver: bridge
    driver_opts:
      com.docker.network.driver.mtu: "1450"
```

## Verify Fix

```bash
# After setting correct MTU:
docker run --rm alpine ping -M do -s 1400 -c 3 8.8.8.8
# Should succeed

# Test with actual application:
docker run --rm alpine wget -O /dev/null http://speedtest.example.com/largefile
# Should complete without hanging

# Check no fragmentation occurring from containers:
# On the host:
tcpdump -i docker0 -n '(ip[6:2] & 0x3fff) != 0'
# Should see no fragmentation if MTU is correct
```

## Conclusion

Container MTU issues arise when the container's network interface MTU doesn't account for overlay encapsulation overhead. Always calculate container MTU as: `host_path_mtu - overlay_overhead` where VXLAN adds 50 bytes and WireGuard adds ~80 bytes. Configure Docker daemon MTU in `/etc/docker/daemon.json` for all new bridges. For Kubernetes, configure the CNI plugin's MTU setting. Test from inside containers with `ping -M do` and check that large file downloads work. Fragmentation in container networks significantly impacts performance and can cause mysterious application timeouts.
