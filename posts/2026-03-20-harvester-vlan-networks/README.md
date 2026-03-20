# How to Set Up VLAN Networks in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, VLAN, Networking

Description: Learn how to configure 802.1Q VLAN networks in Harvester to provide network isolation and segmentation for virtual machine workloads.

## Introduction

VLANs (Virtual Local Area Networks) are a fundamental networking primitive for isolating traffic between different VM workloads, tenants, or security zones. Harvester supports 802.1Q VLAN tagging, allowing VMs to connect to specific VLANs on your physical switching infrastructure. This guide walks through the complete configuration from physical switch setup to VM network attachment.

## Prerequisites

- A Harvester cluster with at least one secondary NIC dedicated to VM traffic
- A managed switch with VLAN support (802.1Q)
- VLANs pre-configured on the switch with the appropriate trunk port to the Harvester nodes
- Network planning document with VLAN IDs and subnets

## Network Design Example

```
VLAN 10  - Management   10.0.10.0/24
VLAN 100 - Production   10.0.100.0/24
VLAN 200 - Staging      10.0.200.0/24
VLAN 300 - DMZ          10.0.300.0/24
```

## Step 1: Configure the Physical Switch

Configure the switch port connected to each Harvester node as a trunk port:

```
! Cisco IOS example - configure trunk port for Harvester nodes
interface GigabitEthernet0/1
  description Harvester-Node-01-eth1
  switchport mode trunk
  switchport trunk allowed vlan 10,100,200,300
  switchport trunk native vlan 1
  no shutdown
```

```
# Linux bridge example (if using a software switch)
# Create bridge and VLAN filtering
ip link add br0 type bridge
ip link set br0 type bridge vlan_filtering 1

# Add the physical NIC to the bridge
ip link set eth1 master br0

# Allow VLANs on the bridge port
bridge vlan add vid 100 dev eth1
bridge vlan add vid 200 dev eth1
bridge vlan add vid 300 dev eth1
```

## Step 2: Create a ClusterNetwork for VLAN Traffic

The ClusterNetwork ties a set of physical NICs across all nodes to a logical network name:

```yaml
# cluster-network-vlan.yaml
# Cluster-wide network backed by eth1 on all nodes

apiVersion: network.harvesterhci.io/v1beta1
kind: ClusterNetwork
metadata:
  name: vlan
spec:
  description: "VLAN-capable network using eth1"
  enable: true
  mtu: 9000  # Enable jumbo frames for performance
```

```bash
kubectl apply -f cluster-network-vlan.yaml
```

## Step 3: Configure NodeNetwork for Each Node

Each node needs a NodeNetwork resource that maps the ClusterNetwork to a physical NIC:

```bash
# Create NodeNetwork for each node using a script
for NODE in harvester-node-01 harvester-node-02 harvester-node-03; do
kubectl apply -f - <<EOF
apiVersion: network.harvesterhci.io/v1beta1
kind: NodeNetwork
metadata:
  name: ${NODE}-vlan
  namespace: harvester-system
spec:
  nodeName: ${NODE}
  nic: eth1
  clusterNetwork: vlan
EOF
done
```

Verify the NodeNetwork is applied successfully:

```bash
kubectl get nodenetwork -n harvester-system
# All nodes should show READY: True
```

## Step 4: Create VLAN Networks via the UI

1. Navigate to **Networks** → **VM Networks**
2. Click **Create**

For VLAN 100 (Production):
```
Name:            prod-vlan-100
Cluster Network: vlan
VLAN ID:         100
Namespace:       default
```

Repeat for each VLAN:
```
Name:            staging-vlan-200
Cluster Network: vlan
VLAN ID:         200
Namespace:       default

Name:            dmz-vlan-300
Cluster Network: vlan
VLAN ID:         300
Namespace:       default
```

## Step 5: Create VLAN Networks via kubectl

```yaml
# prod-vlan-100.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: prod-vlan-100
  namespace: default
  labels:
    network.harvesterhci.io/clusternetwork: vlan
  annotations:
    network.harvesterhci.io/route: '{"mode":"auto"}'
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "prod-vlan-100",
      "type": "bridge",
      "bridge": "mgmt-br",
      "promiscMode": true,
      "vlan": 100,
      "ipam": {}
    }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: staging-vlan-200
  namespace: default
  labels:
    network.harvesterhci.io/clusternetwork: vlan
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "staging-vlan-200",
      "type": "bridge",
      "bridge": "mgmt-br",
      "promiscMode": true,
      "vlan": 200,
      "ipam": {}
    }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: dmz-vlan-300
  namespace: default
  labels:
    network.harvesterhci.io/clusternetwork: vlan
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "dmz-vlan-300",
      "type": "bridge",
      "bridge": "mgmt-br",
      "promiscMode": true,
      "vlan": 300,
      "ipam": {}
    }
```

```bash
kubectl apply -f prod-vlan-100.yaml
```

## Step 6: Attach a VM to a VLAN Network

```yaml
# vm-in-production-vlan.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: prod-web-01
  namespace: default
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 4
        resources:
          requests:
            memory: 8Gi
        machine:
          type: q35
        devices:
          interfaces:
            # Management interface for cluster access
            - name: default
              model: virtio
              masquerade: {}
            # Production VLAN interface
            - name: prod-net
              model: virtio
              bridge: {}
      networks:
        - name: default
          pod: {}
        - name: prod-net
          multus:
            networkName: default/prod-vlan-100  # References the NetworkAttachmentDefinition
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: prod-web-01-root
        - name: cloudinit
          cloudInitNoCloud:
            userData: |
              #cloud-config
              # Configure eth1 for VLAN 100
              write_files:
                - path: /etc/netplan/60-prod-vlan.yaml
                  content: |
                    network:
                      version: 2
                      ethernets:
                        enp2s0:
                          dhcp4: true
              runcmd:
                - netplan apply
```

## Step 7: Verify VLAN Connectivity

```bash
# Access the VM console
virtctl console prod-web-01 -n default

# Inside the VM, verify both interfaces are up
ip addr show

# Expected output:
# enp1s0: 10.42.x.x (management - pod network)
# enp2s0: 10.0.100.x (VLAN 100 - from DHCP or static)

# Test VLAN reachability
ping -c 3 10.0.100.1   # VLAN 100 gateway

# Verify VLAN tagging by checking the bridge
# (on the Harvester host)
bridge vlan show dev mgmt-br
```

## VLAN Isolation Testing

```bash
# Verify that VLAN 100 VMs cannot reach VLAN 200 VMs directly
# (should be blocked by switch/router unless explicitly routed)

# From a VM in VLAN 100:
ping -c 3 10.0.200.x  # Should fail if VLANs are properly isolated

# From a VM in VLAN 100:
ping -c 3 10.0.100.x  # Should succeed (same VLAN)
```

## Conclusion

VLAN networks in Harvester provide a powerful mechanism for network isolation without requiring separate physical infrastructure. By leveraging 802.1Q tagging and Multus CNI, you can connect VMs to specific network segments that match your security and operational requirements. The declarative Kubernetes approach means VLAN configurations can be templated and deployed consistently across environments. Combine VLANs with Kubernetes NetworkPolicies and external firewall rules for a defense-in-depth network security strategy.
