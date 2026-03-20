# How to Configure Kata Containers with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kata Containers, IPv6, Container Runtime, Security, Virtualization, CNI

Description: A guide to configuring Kata Containers with IPv6 networking, covering the unique networking model of VM-based containers and CNI plugin integration.

Kata Containers runs each container (or pod) inside a lightweight VM for stronger isolation. The networking model differs from standard containers: a veth pair bridges the host CNI network into the VM's network namespace, so IPv6 configuration must work correctly both at the host CNI layer and inside the Kata VM.

## How Kata Containers Networking Works

```
Host Network Namespace
    │
    ├── CNI plugin creates veth pair + assigns IPv6 address
    │       tap0 (host side) ←──────→ eth0 (inside Kata VM)
    │
Kata VM (qemu/firecracker/cloud-hypervisor)
    └── Container process sees eth0 with CNI-assigned IPv6
```

The Kata runtime (`kata-runtime`) passes CNI-provided network configuration into the VM via a network hotplug mechanism. IPv6 addresses assigned by CNI appear correctly inside the Kata VM.

## Installation and CNI Setup

```bash
# Install Kata Containers (Ubuntu)
bash -c "$(curl -fsSL https://raw.githubusercontent.com/kata-containers/kata-containers/main/utils/kata-manager.sh) install-kata-tools"

# Or via snap
snap install kata-containers --classic

# Verify installation
kata-runtime check
kata-runtime --version
```

## CNI Configuration with IPv6 for Kata

```json
// /etc/cni/net.d/10-kata-ipv6.conflist

{
  "cniVersion": "1.0.0",
  "name": "kata-net",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "kata0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.88.0.0/16"}],
          [{"subnet": "fd00:kata::/64"}]
        ],
        "routes": [
          {"dst": "0.0.0.0/0"},
          {"dst": "::/0"}
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
```

## Using Kata with containerd and IPv6

```toml
# /etc/containerd/config.toml — add Kata runtime class

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata.options]
    ConfigPath = "/opt/kata/share/defaults/kata-containers/configuration.toml"
```

```bash
# Run a container using Kata runtime via nerdctl
nerdctl run -d \
  --name web \
  --runtime io.containerd.kata.v2 \
  --network kata-net \
  nginx:alpine

# Verify container got IPv6 address
nerdctl exec web ip -6 addr show eth0

# Verify you're inside a VM (Kata shows kata-agent as PID 1 inside VM)
nerdctl exec web ls /proc/1/exe
```

## Kata Containers with Kubernetes

```yaml
# RuntimeClass for Kata
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-containers
handler: kata
---
# Pod using Kata runtime
apiVersion: v1
kind: Pod
metadata:
  name: secure-web
spec:
  runtimeClassName: kata-containers
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
```

```bash
# Apply and verify
kubectl apply -f secure-pod.yaml
kubectl get pod secure-web -o wide

# Check IPv6 address from inside the Kata VM
kubectl exec secure-web -- ip -6 addr show

# Verify isolation (should show kata-agent or hypervisor process namespace)
kubectl exec secure-web -- cat /proc/cpuinfo | grep "model name" | head -1
```

## Kata Configuration for IPv6

```toml
# /opt/kata/share/defaults/kata-containers/configuration.toml

[hypervisor.qemu]
path = "/opt/kata/bin/qemu-system-x86_64"
kernel = "/opt/kata/share/kata-containers/vmlinuz.container"
image = "/opt/kata/share/kata-containers/kata-containers.img"

# Networking: Kata uses tap devices for VM network interface
# IPv6 is handled by the guest kernel — no special Kata-side config needed

[runtime]
# Enable debug for networking issues
enable_debug = false
```

## Troubleshooting Kata IPv6 Networking

```bash
# Check if veth/tap device got IPv6 from CNI
ip -6 addr show | grep kata

# Check Kata VM hypervisor process
ps aux | grep qemu | grep "kata"

# Check kata-agent logs (inside VM logs)
journalctl -u containerd | grep "kata\|ipv6" | tail -30

# Run a Kata container interactively to debug network
nerdctl run --rm -it \
  --runtime io.containerd.kata.v2 \
  ubuntu:22.04 bash

# Inside the container (actually inside a Kata VM):
ip -6 addr show
ip -6 route show
ping6 -c 3 fd00:kata::1   # Ping the gateway
```

## IPv6 Firewall Considerations for Kata

Since Kata containers run inside VMs, the host iptables/ip6tables rules affect traffic at the veth/tap level. Standard CNI masquerade rules apply:

```bash
# Verify ip6tables masquerade is set for Kata network bridge
ip6tables -t nat -L POSTROUTING -n | grep kata0

# If missing, add masquerade for outbound IPv6 from Kata containers
ip6tables -t nat -A POSTROUTING -s fd00:kata::/64 ! -d fd00:kata::/64 -j MASQUERADE

# Allow ICMPv6 through (required for NDP inside VM)
ip6tables -A FORWARD -i kata0 -p ipv6-icmp -j ACCEPT
ip6tables -A FORWARD -o kata0 -p ipv6-icmp -j ACCEPT
```

Kata Containers transparently passes CNI-assigned IPv6 addresses into the lightweight VM, so workloads see the expected IPv6 network interfaces. The key is ensuring the CNI plugin correctly assigns IPv6 addresses and that ICMPv6/NDP is not blocked at the host network layer.
