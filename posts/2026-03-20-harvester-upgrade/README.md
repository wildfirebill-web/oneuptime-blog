# How to Upgrade Harvester

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Harvester, Kubernetes, Virtualization, HCI, Upgrade, Maintenance

Description: A step-by-step guide to upgrading your Harvester HCI cluster to a new version with minimal downtime using rolling upgrades.

## Introduction

Harvester supports rolling upgrades that update nodes one at a time, allowing VMs to continue running during the upgrade process through live migration. The upgrade process updates the Harvester OS, RKE2 Kubernetes version, and all system components including Longhorn, KubeVirt, and the Harvester UI. Proper planning and pre-upgrade checks are essential to ensure a smooth upgrade.

## Upgrade Process Overview

```mermaid
graph LR
    A[Pre-upgrade checks] --> B[Download upgrade image]
    B --> C[Upgrade node 1\nlive migrate VMs off]
    C --> D[Upgrade node 2\nlive migrate VMs off]
    D --> E[Upgrade node 3\nlive migrate VMs off]
    E --> F[Upgrade system components]
    F --> G[Verification]
```

## Step 1: Pre-Upgrade Checklist

Before starting the upgrade, verify cluster health:

```bash
# Check all nodes are Ready

kubectl get nodes

# Check no nodes are cordoned or in maintenance mode
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.unschedulable}{"\n"}{end}'

# Verify all Harvester system pods are running
kubectl get pods -n harvester-system

# Check Longhorn is healthy
kubectl get volumes.longhorn.io -n longhorn-system \
    -o jsonpath='{range .items[*]}{.metadata.name}: {.status.state}{"\n"}{end}'

# Ensure all volumes have sufficient replicas
kubectl get volumes.longhorn.io -n longhorn-system \
    -o jsonpath='{range .items[*]}{.metadata.name}: robustness={.status.robustness}{"\n"}{end}'
# All should show: robustness=healthy

# Check there are no degraded volumes
kubectl get volumes.longhorn.io -n longhorn-system | grep -v healthy
```

```bash
# Check current Harvester version
kubectl get setting server-version -n harvester-system \
    -o jsonpath='{.value}'

# Check available upgrades (from Harvester UI or GitHub releases)
# https://github.com/harvester/harvester/releases
```

## Step 2: Create VM and Data Backups

Before any upgrade, back up critical VMs:

```bash
# Create a backup of each critical VM
for VM in $(kubectl get vm -n default -o name | sed 's|virtualmachine.kubevirt.io/||'); do
    echo "Creating backup for: ${VM}"
    kubectl apply -f - <<EOF
apiVersion: harvesterhci.io/v1beta1
kind: VirtualMachineBackup
metadata:
  name: ${VM}-pre-upgrade-$(date +%Y%m%d)
  namespace: default
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: ${VM}
  type: backup
EOF
done

# Wait for all backups to complete
kubectl get virtualmachinebackup -n default -w
```

## Step 3: Check Upgrade Compatibility

```bash
# Check the Harvester upgrade documentation for the specific version
# Key compatibility items to verify:
# 1. RKE2 version compatibility
# 2. Rancher version compatibility (if integrated)
# 3. Node hardware requirements
# 4. Network compatibility

# For Rancher-integrated clusters, check Rancher support matrix
# https://www.suse.com/suse-harvester/support-matrix/all-supported-versions/
```

## Step 4: Initiate the Upgrade via the UI

1. Log in to the Harvester dashboard
2. Navigate to **Dashboard**
3. If a new version is available, an upgrade banner appears
4. Click **Upgrade Now**
5. Select the upgrade version from the dropdown
6. Click **Start Upgrade**

## Step 5: Initiate the Upgrade via kubectl

```yaml
# harvester-upgrade.yaml
# Trigger a Harvester upgrade to a specific version

apiVersion: harvesterhci.io/v1beta1
kind: Upgrade
metadata:
  name: hvst-upgrade-v1-3-0
  namespace: harvester-system
spec:
  # Target Harvester version
  version: v1.3.0
  # Optional: use a specific image URL
  # image: ""
```

```bash
kubectl apply -f harvester-upgrade.yaml

# Watch the upgrade progress
kubectl get upgrade -n harvester-system -w

# Get detailed upgrade status
kubectl describe upgrade hvst-upgrade-v1-3-0 -n harvester-system
```

## Step 6: Monitor the Upgrade Progress

```bash
# The upgrade creates an "UpgradeLog" that captures all events
kubectl get upgradelogs -n harvester-system

# Follow the upgrade log
kubectl logs -n harvester-system \
    $(kubectl get pods -n harvester-system -l app=upgrade-log -o name) \
    --follow

# Check which nodes have been upgraded
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion,STATUS:.status.conditions[-1].type'

# Watch node upgrades
watch kubectl get nodes
```

During the upgrade, each node goes through these phases:
1. **VMs migrate off**: Running VMs are live-migrated to other nodes
2. **Node drains**: No new workloads scheduled
3. **OS upgrade**: Harvester OS is updated
4. **Node reboots**: Node restarts with new software
5. **Node rejoins**: Node rejoins the cluster
6. **VMs migrate back**: VMs can schedule on the upgraded node

## Step 7: Handle Upgrade Issues

```bash
# If the upgrade is stuck, check the upgrade job status
kubectl get jobs -n harvester-system | grep upgrade

# Check upgrade pod logs
kubectl logs -n harvester-system \
    $(kubectl get pods -n harvester-system -l app=upgrade -o name)

# If a node fails to upgrade:
# 1. Check the node's status
kubectl describe node harvester-node-02

# 2. Check system logs on the node
ssh rancher@192.168.1.12
journalctl -xe --unit=rke2-server
journalctl -xe --unit=upgrade

# 3. If the node is stuck in maintenance mode, forcibly uncordon
kubectl uncordon harvester-node-02
```

## Step 8: Post-Upgrade Verification

After all nodes complete the upgrade:

```bash
# Verify all nodes are on the new version
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion'

# Check Harvester system version
kubectl get setting server-version -n harvester-system \
    -o jsonpath='{.value}'

# Verify all system pods are running
kubectl get pods -n harvester-system
kubectl get pods -n longhorn-system
kubectl get pods -n cattle-system

# Check all VMs are running
kubectl get vmi -A

# Run the verification checklist
echo "=== Post-Upgrade Verification ==="
echo "Node count: $(kubectl get nodes --no-headers | wc -l)"
echo "Running VMs: $(kubectl get vmi -A --no-headers | grep Running | wc -l)"
echo "Healthy volumes: $(kubectl get volumes.longhorn.io -n longhorn-system \
    --no-headers | grep healthy | wc -l)"
echo "System pods (running): $(kubectl get pods -n harvester-system \
    --no-headers | grep Running | wc -l)"
```

## Rollback Considerations

Harvester does not support in-place rollback after a successful upgrade. Before upgrading:
- Back up all VMs to an external backup target
- Export VM disks that are critical
- Document the current configuration

If an upgrade fails partway through:
```bash
# Check the upgrade status for failure details
kubectl get upgrade -n harvester-system -o yaml | grep -A 10 "status:"

# Contact SUSE/Harvester support with upgrade logs
kubectl logs -n harvester-system \
    $(kubectl get pods -n harvester-system -l app=upgrade -o name) > upgrade-debug.log
```

## Conclusion

Upgrading Harvester is designed to be a minimally disruptive process thanks to live migration of VMs during node upgrades. The rolling upgrade approach ensures that your VM workloads continue running throughout the entire upgrade process. Always perform pre-upgrade health checks, create backups of critical VMs, and monitor the upgrade progress closely. Testing the upgrade process in a staging environment before applying it to production is strongly recommended for major version upgrades.
