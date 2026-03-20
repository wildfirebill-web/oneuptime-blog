# How to Set Up Storage Networks in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Storage, Longhorn, Networking

Description: Configure dedicated storage networks in Harvester to separate Longhorn replication traffic from management and VM networks for optimal performance.

## Introduction

By default, Harvester uses the management network for all traffic, including Longhorn's storage replication between nodes. In production environments, storage replication traffic can consume significant bandwidth, potentially impacting management operations and VM performance. Setting up a dedicated storage network isolates this traffic, ensuring predictable performance for all workloads.

## Why a Dedicated Storage Network?

```
Without dedicated storage network:
  Management traffic + Storage replication + API traffic = Congested management NIC

With dedicated storage network:
  Management NIC: API traffic, Kubernetes control plane (low bandwidth)
  Storage NIC:    Longhorn replication traffic (high bandwidth, sustained)
  VM NIC:         Guest VM traffic (variable)
```

## Prerequisites

- Each Harvester node needs a third NIC (in addition to management and VM NICs)
- A dedicated switch (or VLAN) for storage traffic
- Storage NICs should be identical across all nodes for consistent performance
- Recommended: 10 GbE or 25 GbE for storage networks

## Step 1: Plan the Storage Network

```
Storage Network CIDR: 10.200.0.0/24
Node 1 Storage IP:    10.200.0.11
Node 2 Storage IP:    10.200.0.12
Node 3 Storage IP:    10.200.0.13
MTU:                  9000 (jumbo frames for storage performance)
NIC:                  eth2 on each node
```

## Step 2: Configure Storage NICs on Each Node

SSH into each node and configure the storage NIC:

```bash
# SSH into Node 1
ssh rancher@192.168.1.11

# Check available NICs
ip link show

# Configure the storage NIC with a static IP
sudo vi /etc/sysconfig/network/ifcfg-eth2
```

```ini
# /etc/sysconfig/network/ifcfg-eth2
# Storage network interface configuration

STARTMODE='auto'
BOOTPROTO='static'
IPADDR='10.200.0.11'
NETMASK='255.255.255.0'
# No default gateway on storage NIC (traffic stays local)
# MTU for jumbo frames
MTU='9000'
```

```bash
# Bring up the storage interface
sudo wicked ifup eth2

# Verify the interface is up with the correct IP
ip addr show eth2

# Test connectivity to other nodes
ping -c 3 10.200.0.12
ping -c 3 10.200.0.13
```

Repeat on Node 2 (10.200.0.12) and Node 3 (10.200.0.13).

## Step 3: Configure Longhorn to Use the Storage Network

Longhorn uses the `storage-network` setting to direct replication traffic to a specific network interface:

### Via the Harvester UI

1. Navigate to **Settings** → **Harvester Settings**
2. Find **Storage Network** and click **Edit**
3. Enter the storage network CIDR: `10.200.0.0/24`
4. Click **Save**

### Via kubectl

```yaml
# longhorn-storage-network.yaml
# Configure Longhorn to use the dedicated storage network

apiVersion: longhorn.io/v1beta2
kind: Setting
metadata:
  name: storage-network
  namespace: longhorn-system
spec:
  # CIDR of the storage network
  # Longhorn will use the NIC with an IP in this range for replication
  value: "10.200.0.0/24"
```

```bash
kubectl apply -f longhorn-storage-network.yaml

# Verify the setting was applied
kubectl get setting storage-network -n longhorn-system \
    -o jsonpath='{.spec.value}'
```

### Verify Longhorn Is Using the Storage Network

After applying the setting, Longhorn pods will restart. Verify the network is being used:

```bash
# Check Longhorn manager logs for storage network information
kubectl logs -n longhorn-system \
    $(kubectl get pods -n longhorn-system -l app=longhorn-manager -o name | head -1) \
    | grep -i "storage network"

# Check the node's storage IP is visible to Longhorn
kubectl get nodes.longhorn.io -n longhorn-system -o yaml | \
    grep -A 5 "storageIP"
```

## Step 4: Configure MTU for Jumbo Frames

Jumbo frames (MTU 9000) significantly improve storage throughput by reducing CPU overhead:

```bash
# Verify the switch/physical layer supports jumbo frames
# (The switch port MTU must also be set to 9000)

# Check current MTU on all nodes
for NODE in 192.168.1.11 192.168.1.12 192.168.1.13; do
    echo -n "${NODE} eth2 MTU: "
    ssh rancher@${NODE} "ip link show eth2 | grep mtu"
done

# Test jumbo frame connectivity between nodes
# (packet size 8972 = 9000 - 28 bytes for IP+ICMP headers)
ping -M do -s 8972 -c 3 10.200.0.12
```

## Step 5: Separate Storage Traffic with Network Policies

Additionally, add Kubernetes NetworkPolicies to ensure no VM traffic accidentally routes through the storage network:

```yaml
# storage-network-policy.yaml
# Restrict storage namespace traffic to only Longhorn pods

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: longhorn-storage-isolation
  namespace: longhorn-system
spec:
  # Apply to all Longhorn pods
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only allow traffic from Longhorn pods and Harvester system
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: longhorn-system
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: harvester-system
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: longhorn-system
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: harvester-system
```

## Step 6: Monitor Storage Network Performance

```bash
# Monitor storage network utilization on each node
# SSH into a node and run:
sar -n DEV 1 10 | grep eth2

# Or use iftop for real-time monitoring
iftop -i eth2

# Check Longhorn replication status (should see storage IPs in use)
kubectl get replicas -n longhorn-system -o wide

# Verify no storage traffic is flowing through management NIC
iftop -i eth0  # Should show minimal traffic during storage operations
```

## Benchmarking Storage Throughput

After configuring the storage network, benchmark to verify improvement:

```bash
# Run a Longhorn disk benchmark
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: storage-benchmark
  namespace: default
spec:
  containers:
    - name: fio
      image: nixery.dev/fio
      command: ["fio", "--name=randwrite", "--ioengine=libaio",
                "--iodepth=16", "--rw=randwrite", "--bs=4k",
                "--direct=1", "--size=1G", "--numjobs=4",
                "--runtime=60", "--time_based", "--group_reporting",
                "--filename=/mnt/test/testfile"]
      volumeMounts:
        - mountPath: /mnt/test
          name: test-vol
  volumes:
    - name: test-vol
      persistentVolumeClaim:
        claimName: benchmark-pvc
  restartPolicy: Never
EOF
```

## Conclusion

A dedicated storage network is an essential optimization for production Harvester deployments. By directing Longhorn replication traffic to a separate high-bandwidth NIC — ideally with jumbo frames enabled — you eliminate a significant source of network contention. The result is more predictable latency for management operations, better VM network performance, and improved Longhorn replication throughput. Implement the storage network before adding significant VM workloads to avoid disruptive reconfigurations later.
