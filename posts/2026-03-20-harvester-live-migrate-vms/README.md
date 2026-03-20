# How to Live Migrate VMs in Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Live Migration, KubeVirt

Description: A detailed guide to live migrating virtual machines in Harvester with zero downtime, covering prerequisites, configuration, and troubleshooting.

## Introduction

Live migration transfers a running virtual machine from one physical host to another without interrupting the VM's operation. During migration, the VM's memory is copied to the destination node while the VM continues to run. Once the memory copy is nearly complete, the VM briefly pauses (typically milliseconds to seconds), completes the final memory transfer, and resumes on the destination node. This results in zero or near-zero downtime for the VM's workloads.

## How Live Migration Works

```mermaid
graph LR
    A["Source Node\nVM Running"] --> B["Memory Pre-copy\n(VM still running on source)"]
    B --> C["Final Stop-and-Copy\n(brief pause)"]
    C --> D["VM Resumes on Target"]
    D --> E["Source Resources Released"]
```

The migration process:
1. A migration pod starts on the target node
2. VM memory is pre-copied to the target (iteratively, starting with dirty pages)
3. When the dirty page rate drops below a threshold, a final stop-and-copy occurs
4. The VM resumes on the target node
5. Source VM resources are released

## Prerequisites for Live Migration

Live migration requires:
- At least 2 nodes with sufficient resources
- Shared storage (Longhorn - already built into Harvester)
- Network connectivity between nodes for memory transfer
- The VM must not have PCI passthrough or SR-IOV devices (these prevent live migration)

## Step 1: Verify Live Migration Capability

```bash
# Check if a VM is migratable

kubectl get vmi ubuntu-web-01 -n default \
    -o jsonpath='{.status.conditions[?(@.type=="LiveMigratable")]}' | jq .

# If reason is "True", the VM can be live migrated
# If reason contains "NonMigratable", see the message for why
```

Common reasons a VM cannot be live migrated:
- Has host devices (PCI passthrough)
- Uses SR-IOV interfaces
- Has CPU pinning configured without appropriate settings
- Has local volumes not backed by a shared storage class

## Step 2: Configure Live Migration Bandwidth

Control migration bandwidth to avoid impacting running workloads:

```yaml
# kubevirt-migration-config.yaml
# Configure global migration settings

apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: harvester-system
spec:
  configuration:
    migrations:
      # Maximum bandwidth for a single migration (bytes/sec)
      # 64 Mi = 64 MB/s - conservative setting to avoid network saturation
      bandwidthPerMigration: "64Mi"
      # Maximum number of concurrent migrations per cluster
      parallelMigrationsPerCluster: 5
      # Maximum migrations per node
      parallelOutboundMigrationsPerNode: 2
      # Timeout in seconds for the final migration phase
      completionTimeoutPerGiB: 800
      # Maximum time for the entire migration (seconds)
      progressTimeout: 150
      # Allow post-copy migration as a fallback
      allowPostCopy: false
      # Network for migration traffic
      network: ""
```

```bash
kubectl apply -f kubevirt-migration-config.yaml
```

## Step 3: Perform a Live Migration

### Via the UI

1. Navigate to **Virtual Machines**
2. Find the running VM
3. Click the **⋮** menu → **Migrate**
4. The migration starts immediately (auto-selects target node)

### Via kubectl

```yaml
# live-migration.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceMigration
metadata:
  name: live-mig-ubuntu-web-01
  namespace: default
spec:
  vmiName: ubuntu-web-01
```

```bash
kubectl apply -f live-migration.yaml

# Track detailed migration progress
kubectl get virtualmachineinstancemigration live-mig-ubuntu-web-01 \
    -n default -o yaml | grep -A 30 "status:"
```

### Using virtctl

```bash
# Migrate using virtctl
virtctl migrate ubuntu-web-01 -n default

# Check migration state
virtctl migration-info ubuntu-web-01 -n default
```

## Step 4: Monitor Migration Progress

```bash
# Watch migration status
watch -n 2 kubectl get vmsimigration -n default

# Get migration memory transfer progress
kubectl get vmi ubuntu-web-01 -n default \
    -o jsonpath='{.status.migrationState}' | jq '{
        mode: .mode,
        startTimestamp: .startTimestamp,
        endTimestamp: .endTimestamp,
        targetNodeAddress: .targetNodeAddress,
        targetNode: .targetNode,
        sourceNode: .sourceNode
    }'

# Watch the migration bandwidth consumption
# SSH into a node and monitor network
sar -n DEV 1 60 | grep eth0
```

## Step 5: Set Up a Dedicated Migration Network

For production clusters, use a dedicated network for migration traffic to avoid impacting VM networking:

```yaml
# migration-network.yaml
# Dedicated migration network configuration

apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-config
  namespace: harvester-system
data:
  # Configure migrations to use a specific network
  # This is the name of a Multus NetworkAttachmentDefinition
  migrations: |
    {
      "network": "default/migration-network",
      "bandwidthPerMigration": "128Mi"
    }
```

Create the migration network:

```yaml
# migration-nad.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: migration-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "migration-network",
      "type": "macvlan",
      "master": "eth2",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "10.201.0.0/24",
        "rangeStart": "10.201.0.10",
        "rangeEnd": "10.201.0.200"
      }
    }
```

## Step 6: Cancel a Migration

```bash
# Cancel a running migration
kubectl delete virtualmachineinstancemigration live-mig-ubuntu-web-01 -n default

# The VM will continue running on the source node
kubectl get vmi ubuntu-web-01 -n default \
    -o jsonpath='{.status.nodeName}'
```

## Step 7: Validate After Migration

```bash
# Verify VM is running on the new node
kubectl get vmi ubuntu-web-01 -n default \
    -o custom-columns='NAME:.metadata.name,NODE:.status.nodeName,PHASE:.status.phase'

# Check VM responsiveness
VM_IP=$(kubectl get vmi ubuntu-web-01 -n default \
    -o jsonpath='{.status.interfaces[0].ipAddress}')

# Ping the VM
ping -c 5 $VM_IP

# Test application availability
curl -sf http://$VM_IP/healthz && echo "App healthy after migration"

# Verify no memory corruption (check VM logs)
virtctl console ubuntu-web-01 -n default
```

## Performance Tuning for Large Memory VMs

For VMs with large amounts of RAM (e.g., 256 GB), migrations can take a long time:

```bash
# Check how long the last migration took
kubectl get vmi ubuntu-web-01 -n default \
    -o jsonpath='{.status.migrationState}' | jq '{
        start: .startTimestamp,
        end: .endTimestamp,
        mode: .mode
    }'

# For large VM migrations, increase the timeout
kubectl patch kubevirt kubevirt -n harvester-system --type json \
-p '[{
    "op": "replace",
    "path": "/spec/configuration/migrations/completionTimeoutPerGiB",
    "value": 1200
}]'

# Increase bandwidth limit for faster migration
kubectl patch kubevirt kubevirt -n harvester-system --type json \
-p '[{
    "op": "replace",
    "path": "/spec/configuration/migrations/bandwidthPerMigration",
    "value": "256Mi"
}]'
```

## Conclusion

Live migration is one of Harvester's most valuable features for maintaining service availability during infrastructure operations. By transferring running VMs between nodes without downtime, you can perform hardware maintenance, apply security patches to nodes, and rebalance workloads without impacting users. The key to successful live migrations is proper planning: adequate network bandwidth, sufficient resources on target nodes, and a dedicated migration network to prevent interference with production traffic.
