# How to Configure Harvester with Multiple NICs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Networking, Multiple NICs, Bonding

Description: Learn how to configure Harvester nodes with multiple NICs for dedicated management, storage, and VM networks for optimal performance and isolation.

## Introduction

Production Harvester deployments benefit greatly from multiple NICs per node. Each NIC can be dedicated to a specific traffic type: management (cluster control plane + API), storage (Longhorn replication), and VM traffic (guest VM networks). This traffic isolation ensures that heavy storage replication or VM traffic never impacts cluster management operations, and vice versa.

## Recommended Multi-NIC Layout

```
NIC 1 (eth0/bond0): Management Network
  - Kubernetes API
  - etcd replication
  - Harvester UI
  - SSH access

NIC 2 (eth1): Storage Network
  - Longhorn volume replication
  - iSCSI traffic

NIC 3 (eth2/bond1): VM Network
  - Guest VM traffic
  - VLAN trunking
```

## Step 1: Plan Network Layout

Before installation, document your NIC and IP layout:

```
Node 1:
  Management:  192.168.1.11/24  (eth0)
  Storage:     10.200.0.11/24   (eth1)
  VM:          (eth2 - unaddressed, used by VMs only)

Node 2:
  Management:  192.168.1.12/24  (eth0)
  Storage:     10.200.0.12/24   (eth1)
  VM:          (eth2 - unaddressed)

Node 3:
  Management:  192.168.1.13/24  (eth0)
  Storage:     10.200.0.13/24   (eth1)
  VM:          (eth2 - unaddressed)
```

## Step 2: Configure Multiple NICs During Installation

Use a Harvester config file for automated multi-NIC setup:

```yaml
# multi-nic-config.yaml
# Harvester node configuration with 3 NICs

scheme_version: 1

install:
  device: /dev/sda
  automatic: true

os:
  hostname: harvester-node-01
  ssh_authorized_keys:
    - ssh-ed25519 AAAAC3NzaC1... admin@host
  ntp_servers:
    - pool.ntp.org

# Network interface configuration
network:
  interfaces:
    # Management NIC - primary interface
    - name: eth0
      hwAddr: "aa:bb:cc:dd:ee:01"  # Optional: specify MAC
    # Storage NIC
    - name: eth1
      hwAddr: "aa:bb:cc:dd:ee:02"
    # VM NIC (configured separately as a cluster network)
    - name: eth2
      hwAddr: "aa:bb:cc:dd:ee:03"

harvester:
  mode: create
  # Management interface configuration
  management_interface:
    interfaces:
      - name: eth0
    method: static
    ip: 192.168.1.11
    subnetMask: 255.255.255.0
    gateway: 192.168.1.1
    dnsNameservers:
      - 8.8.8.8
      - 8.8.4.4
  token: "my-cluster-token"
  vip: 192.168.1.100
  vip_mode: static
  password: "HarvesterAdmin123!"
```

## Step 3: Configure Bonding for Redundancy

For high availability, bond two NICs together for each network:

```yaml
# bonded-nic-config.yaml
# Configuration with NIC bonding for redundancy

scheme_version: 1

network:
  interfaces:
    - name: eth0  # Management bond member 1
    - name: eth1  # Management bond member 2
    - name: eth2  # Storage NIC
    - name: eth3  # VM NIC 1
    - name: eth4  # VM NIC 2

  # Management bond (active-backup for simplicity)
  bonds:
    - name: harvester-mgmt
      mode: active-backup
      slaves:
        - eth0
        - eth1
      mtu: 1500

harvester:
  management_interface:
    interfaces:
      - name: harvester-mgmt  # Use the bond
    method: static
    ip: 192.168.1.11
    subnetMask: 255.255.255.0
    gateway: 192.168.1.1
```

## Step 4: Configure Storage NIC After Installation

Once the cluster is up, configure the storage NIC on each node:

```bash
# SSH into each node and configure the storage NIC

# On Node 1
ssh rancher@192.168.1.11

# Configure eth1 for storage traffic
sudo vi /etc/sysconfig/network/ifcfg-eth1
```

```ini
# /etc/sysconfig/network/ifcfg-eth1
# Storage network interface

STARTMODE='auto'
BOOTPROTO='static'
IPADDR='10.200.0.11'
NETMASK='255.255.255.0'
MTU='9000'  # Jumbo frames for storage performance
```

```bash
# Bring up the storage interface
sudo wicked ifup eth1

# Verify
ip addr show eth1
ping -c 3 10.200.0.12  # Ping Node 2 storage IP
```

Repeat on all nodes (10.200.0.12 for node 2, 10.200.0.13 for node 3).

## Step 5: Configure Longhorn to Use Storage Network

```yaml
# longhorn-storage-network.yaml
# Direct Longhorn to use the dedicated storage network

apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: storage-network
  namespace: longhorn-system
spec:
  value: "10.200.0.0/24"  # Storage network CIDR
```

```bash
kubectl apply -f longhorn-storage-network.yaml

# Verify Longhorn is using the storage network
kubectl get nodes.longhorn.io -n longhorn-system \
    -o jsonpath='{range .items[*]}{.metadata.name}: {.status.address}{"\n"}{end}'
# Should show 10.200.x.x addresses
```

## Step 6: Configure VM Network NIC

The third NIC (eth2) is used for VM VLAN traffic. Create a ClusterNetwork for it:

```yaml
# vm-cluster-network.yaml
# ClusterNetwork backed by eth2 on all nodes

apiVersion: network.harvesterhci.io/v1beta1
kind: ClusterNetwork
metadata:
  name: vm-network
spec:
  description: "VM network using eth2 on all nodes"
  enable: true
  mtu: 9000
```

```bash
kubectl apply -f vm-cluster-network.yaml

# Configure NodeNetwork for each node to map eth2 to the ClusterNetwork
for NODE in harvester-node-01 harvester-node-02 harvester-node-03; do
kubectl apply -f - <<EOF
apiVersion: network.harvesterhci.io/v1beta1
kind: NodeNetwork
metadata:
  name: ${NODE}-vm
  namespace: harvester-system
spec:
  nodeName: ${NODE}
  nic: eth2
  clusterNetwork: vm-network
EOF
done
```

## Step 7: Verify Traffic Separation

Validate that each type of traffic flows through the correct NIC:

```bash
# Monitor management NIC traffic (should show API, etcd traffic)
iftop -i eth0

# Monitor storage NIC traffic (should show Longhorn replication)
iftop -i eth1
# Trigger volume replication to test:
# Create a test PVC and watch eth1 traffic spike

# Monitor VM NIC (should show VM guest traffic)
iftop -i eth2
# Start a VM and do network transfers to verify

# Check interface statistics
for NIC in eth0 eth1 eth2; do
    echo "=== ${NIC} ==="
    ip -s link show ${NIC} | grep -A 3 "RX:"
done
```

## Step 8: NIC Configuration Persistence

Ensure all NIC configurations persist across reboots:

```bash
# Verify all interfaces are configured in sysconfig
ls /etc/sysconfig/network/ifcfg-*

# Test by rebooting a node
sudo reboot

# After reboot, verify all interfaces are up
ip addr show
# Should show:
# eth0: 192.168.1.11/24 (management)
# eth1: 10.200.0.11/24  (storage)
# eth2: (no IP - used by VMs)

# Verify Longhorn storage network is still active
kubectl get setting storage-network -n longhorn-system \
    -o jsonpath='{.spec.value}'
```

## Performance Impact of Traffic Separation

```
Without traffic separation (all traffic on eth0):
  Management latency:  100-300ms during heavy storage operations
  Storage throughput:  Limited by management traffic sharing NIC
  VM network:          Competing with all other traffic

With dedicated NICs:
  Management latency:  <10ms (dedicated path)
  Storage throughput:  Full NIC bandwidth available (~9 Gbps on 10GbE with jumbo frames)
  VM network:          Predictable performance, not affected by storage operations
```

## Conclusion

Configuring Harvester with multiple dedicated NICs transforms the cluster's performance and reliability. Traffic isolation ensures that heavy Longhorn replication doesn't impact the cluster's control plane responsiveness, and VM network performance remains predictable regardless of storage activity. While a single NIC works for development environments, any production deployment should plan for at least two dedicated NICs: one for management and one for combined storage/VM traffic, with three or more being ideal for full isolation. The investment in additional NICs and switch ports pays dividends in operational stability and performance predictability.
