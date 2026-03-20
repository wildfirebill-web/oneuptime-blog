# How to Configure Longhorn Storage Over Provisioning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Configuration, Capacity Planning

Description: Learn how to configure Longhorn's storage over-provisioning settings to control how volumes are scheduled relative to actual available disk capacity.

## Introduction

Storage over-provisioning allows Longhorn to schedule more volume data than the actual physical disk space available. This works because volumes often do not use their full allocated capacity immediately. For example, a 10 GiB PVC might only contain 2 GiB of actual data. By allowing over-provisioning, you can allocate more total PVC capacity than your physical storage, improving storage utilization efficiency.

## Understanding Over-Provisioning

Longhorn uses two key settings to control storage scheduling:

1. **Storage Over Provisioning Percentage**: How much more than the actual available storage can be scheduled. A value of `200` means you can schedule up to 2x the available storage.

2. **Storage Minimal Available Percentage**: The minimum percentage of storage that must remain free before Longhorn stops scheduling new replicas on a disk. This prevents disks from being completely filled.

### Calculation Example

```text
Available disk space: 100 GiB
Over-provisioning: 200%
Maximum schedulable storage: 100 GiB × 200% = 200 GiB

Minimal available: 25%
Minimum free space: 100 GiB × 25% = 25 GiB
Effective schedulable storage: 175 GiB (200 - 25)
```

## View Current Settings

```bash
# Check current over-provisioning percentage

kubectl get settings.longhorn.io storage-over-provisioning-percentage \
  -n longhorn-system -o yaml

# Check current minimal available percentage
kubectl get settings.longhorn.io storage-minimal-available-percentage \
  -n longhorn-system -o yaml
```

## Configure Over-Provisioning Percentage

### Via kubectl

```bash
# Set over-provisioning to 200% (can schedule 2x available storage)
kubectl patch settings.longhorn.io storage-over-provisioning-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value": "200"}'

# For very efficient clusters with low actual usage, increase to 300%
kubectl patch settings.longhorn.io storage-over-provisioning-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value": "300"}'

# For production clusters with unpredictable usage, reduce to 100% (no over-provisioning)
kubectl patch settings.longhorn.io storage-over-provisioning-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value": "100"}'
```

### Via Longhorn UI

1. Navigate to **Setting** → **General**
2. Find **Storage Over Provisioning Percentage**
3. Set the value
4. Find **Storage Minimal Available Percentage**
5. Set the minimum free space percentage
6. Click **Save**

## Configure Minimal Available Percentage

```bash
# Set minimum 25% free space requirement (recommended)
kubectl patch settings.longhorn.io storage-minimal-available-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value": "25"}'

# For tighter clusters, reduce to 10% (use cautiously)
kubectl patch settings.longhorn.io storage-minimal-available-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value": "10"}'
```

## Configuring with Helm

Set these values during installation or upgrade:

```yaml
# longhorn-values.yaml - Over-provisioning settings
defaultSettings:
  # Allow scheduling up to 200% of available disk space
  storageOverProvisioningPercentage: 200
  # Refuse new replicas when less than 25% disk space remains
  storageMinimalAvailablePercentage: 25
```

```bash
helm upgrade longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --values longhorn-values.yaml
```

## Monitoring Storage Utilization

Monitor disk usage to avoid running out of space:

```bash
# Check disk usage across all nodes
kubectl get nodes.longhorn.io -n longhorn-system \
  -o yaml | grep -A 5 "diskStatus" | grep -E "storageAvailable|storageMaximum|storageScheduled"

# Script to check over-provisioning ratio per node
cat << 'EOF' > check-overprovisioning.sh
#!/bin/bash
echo "=== Longhorn Storage Utilization ==="
kubectl get nodes.longhorn.io -n longhorn-system -o json | \
  jq -r '.items[] |
    .metadata.name as $node |
    .status.diskStatus // {} |
    to_entries[] |
    "\($node)/\(.key): Available=\(.value.storageAvailable // 0) Scheduled=\(.value.storageScheduled // 0) Maximum=\(.value.storageMaximum // 0)"'
EOF
chmod +x check-overprovisioning.sh
./check-overprovisioning.sh
```

## Recommendations by Environment

### Development Environment

```yaml
defaultSettings:
  # High over-provisioning - developers create many small PVCs
  storageOverProvisioningPercentage: 500
  # Low minimum available - acceptable for non-critical workloads
  storageMinimalAvailablePercentage: 10
```

### Production Environment

```yaml
defaultSettings:
  # Conservative over-provisioning - predictable workloads
  storageOverProvisioningPercentage: 150
  # Higher minimum - ensure headroom for replica rebuilds
  storageMinimalAvailablePercentage: 30
```

### Database-Heavy Environment

```yaml
defaultSettings:
  # No over-provisioning - databases actually use their allocated space
  storageOverProvisioningPercentage: 100
  # Higher minimum - databases can grow quickly
  storageMinimalAvailablePercentage: 40
```

## Troubleshooting: Volume Creation Fails Due to Insufficient Storage

If PVC creation fails with insufficient storage errors:

```bash
# Check actual disk availability
kubectl get nodes.longhorn.io -n longhorn-system

# Check what's currently scheduled
kubectl get volumes.longhorn.io -n longhorn-system | awk '{sum += $5} END {print "Total scheduled: " sum " bytes"}'

# Check if over-provisioning limit is reached
kubectl describe nodes.longhorn.io -n longhorn-system | grep -A 10 "Conditions:"
```

Common solutions:
1. Increase over-provisioning percentage temporarily
2. Add more storage disks to nodes
3. Delete unused volumes and PVCs
4. Reduce replica count for non-critical volumes

## Conclusion

Storage over-provisioning is a powerful feature that improves storage utilization efficiency by allowing more volume allocation than actual physical capacity. The right settings depend on your workloads' actual storage usage patterns. Start conservatively and adjust based on observed utilization. Always monitor disk usage and set up alerts before your minimum available threshold is reached to avoid impacting running workloads.
