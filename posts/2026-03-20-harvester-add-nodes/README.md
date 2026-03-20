# How to Add Nodes to Harvester Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Cluster, Scaling

Description: Learn how to expand your Harvester cluster by adding new nodes for increased compute, memory, and storage capacity.

## Introduction

Adding nodes to a Harvester cluster is how you scale capacity - more nodes means more VMs, more CPU and memory, and more distributed storage through Longhorn. The process involves installing Harvester on the new node and joining it to the existing cluster using the cluster token. New nodes automatically join the control plane (Harvester uses RKE2 with each node running etcd and the control plane) and Longhorn begins replicating data to the new node's disks.

## Prerequisites

- An existing Harvester cluster
- The cluster token from the initial installation
- The cluster VIP address
- New server meeting hardware requirements (8+ CPU cores, 32 GB+ RAM, 250 GB+ SSD)
- Harvester ISO on a USB drive or accessible via iPXE

## Step 1: Get the Cluster Join Information

Before installing on the new node, collect the join information from the existing cluster:

```bash
# SSH into any existing cluster node

ssh rancher@192.168.1.11

# Get the cluster token
sudo cat /etc/rancher/rke2/join-token

# Get the cluster VIP (already known, but verify)
kubectl get setting vip-address -n harvester-system \
    -o jsonpath='{.value}' 2>/dev/null || \
    kubectl get service -n harvester-system | grep ingress

# Get the current Harvester version to ensure the new node matches
kubectl get setting server-version -n harvester-system \
    -o jsonpath='{.value}'
```

## Step 2: Download the Matching Harvester ISO

Use the same Harvester version as the existing cluster:

```bash
HARVESTER_VERSION="v1.3.0"  # Match your existing cluster version

# Download the ISO
wget https://releases.rancher.com/harvester/${HARVESTER_VERSION}/harvester-${HARVESTER_VERSION}-amd64.iso

# Write to USB
sudo dd if=harvester-${HARVESTER_VERSION}-amd64.iso of=/dev/sdX bs=4M status=progress
```

## Step 3: Boot the New Node and Select Join Mode

1. Insert the USB into the new server
2. Boot from USB
3. In the Harvester installer, select **Join an existing Harvester cluster**

## Step 4: Configure the New Node

During the join installation wizard:

```text
# Management Network Configuration
Interface:    eth0
Method:       Static
IP Address:   192.168.1.14/24    (next available IP)
Gateway:      192.168.1.1
DNS:          8.8.8.8

# Cluster Join Information
Server URL:   https://192.168.1.100   (cluster VIP)
Cluster Token: <token from Step 1>

# Node Settings
Hostname:     harvester-node-04

# Storage
OS Disk:      /dev/sda  (250 GB SSD)
```

## Step 5: Automated Join with Config File

For consistent node additions, use a configuration file:

```yaml
# join-node-config.yaml
# Configuration for joining a new node to the cluster

scheme_version: 1

install:
  device: /dev/sda
  automatic: true

os:
  hostname: harvester-node-04
  ssh_authorized_keys:
    - ssh-ed25519 AAAAC3NzaC1... admin@host
  password: "$6$rounds=4096$salt$hash"  # Pre-hashed password
  ntp_servers:
    - pool.ntp.org
  dns_nameservers:
    - 8.8.8.8
    - 8.8.4.4

network:
  interfaces:
    - name: eth0
      hwAddr: ""
  bonds:
    - name: harvester-mgmt
      mode: active-backup
      slaves:
        - eth0

harvester:
  # JOIN mode for additional nodes
  mode: join
  management_interface:
    interfaces:
      - name: eth0
    method: static
    ip: 192.168.1.14
    subnetMask: 255.255.255.0
    gateway: 192.168.1.1
    dnsNameservers:
      - 8.8.8.8
  # VIP of the existing cluster
  server_url: https://192.168.1.100
  # Token from the existing cluster
  token: "your-cluster-join-token"
```

## Step 6: Monitor the Node Joining

```bash
# On the existing cluster, watch the new node appear
kubectl get nodes -w

# The new node goes through these states:
# 1. Initializing - RKE2 is starting
# 2. NotReady - Node is up but system pods are deploying
# 3. Ready - Node is fully operational

# Check system pods on the new node
kubectl get pods -A --field-selector spec.nodeName=harvester-node-04

# Verify etcd cluster membership (should now show 4 members)
kubectl exec -n kube-system \
    $(kubectl get pods -n kube-system -l component=etcd -o name | head -1) -- \
    etcdctl --endpoints=https://127.0.0.1:2379 \
            --cacert /var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
            --cert /var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
            --key /var/lib/rancher/rke2/server/tls/etcd/server-client.key \
            member list
```

## Step 7: Verify Longhorn Disk Integration

Once the node joins, Longhorn automatically discovers and adds its disks:

```bash
# Check Longhorn sees the new node
kubectl get nodes.longhorn.io -n longhorn-system

# The new node should appear with its disks
kubectl get disks.longhorn.io -n longhorn-system | grep harvester-node-04

# Wait for volume rebalancing to begin
# Longhorn will start replicating data to the new node's disks
kubectl get volumes.longhorn.io -n longhorn-system \
    -o jsonpath='{range .items[*]}{.metadata.name}: {.status.robustness} replicas={.spec.numberOfReplicas}{"\n"}{end}'
```

## Step 8: Configure Additional Disks on the New Node

If the new node has additional data disks beyond the OS disk:

```bash
# In the Harvester UI:
# 1. Navigate to Hosts
# 2. Click on the new node
# 3. Go to the "Disks" tab
# 4. Click "Add Disk"
# 5. Select the additional disk(s)
# 6. Configure the storage tags if needed
# 7. Click Save

# Via kubectl - tag the disk for Longhorn
kubectl patch node.longhorn.io harvester-node-04 -n longhorn-system \
    --type json \
    -p '[{
        "op": "add",
        "path": "/spec/disks/additional-disk",
        "value": {
            "path": "/var/lib/harvester/defaultdisk/disk2",
            "allowScheduling": true,
            "storageReserved": 10737418240,
            "tags": ["ssd"]
        }
    }]'
```

## Step 9: Post-Join Validation

```bash
# Complete validation checklist

echo "=== Node Join Validation ==="

# 1. Node is Ready
kubectl get node harvester-node-04 | grep Ready

# 2. System pods are running on new node
kubectl get pods -A --field-selector spec.nodeName=harvester-node-04 \
    --no-headers | grep -v Running

# 3. Longhorn disk is schedulable
kubectl get disk -n longhorn-system | grep harvester-node-04

# 4. Try scheduling a test VM on the new node
kubectl apply -f - <<EOF
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: node-test-vm
  namespace: default
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: harvester-node-04
      domain:
        cpu:
          cores: 1
        resources:
          requests:
            memory: 512Mi
        machine:
          type: q35
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          containerDisk:
            image: kubevirt/cirros-registry-disk-demo
EOF

kubectl get vmi node-test-vm -n default -w
kubectl delete vm node-test-vm -n default
```

## Conclusion

Adding nodes to a Harvester cluster is a straightforward process that scales both compute and storage capacity simultaneously. The cluster automatically balances VM scheduling and Longhorn storage replication across the new node. For production clusters, add nodes in groups to maintain odd-numbered control plane counts (3, 5, 7) which is required for etcd quorum. New nodes are available for VM scheduling immediately after joining, and storage rebalancing happens automatically in the background.
