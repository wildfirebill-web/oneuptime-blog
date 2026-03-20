# How to Manage Elemental Machine Inventory

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, Machine Inventory, Edge, Rancher

Description: A guide to managing the Elemental MachineInventory, including viewing, labeling, filtering, and maintaining registered bare metal nodes.

## Introduction

The Elemental MachineInventory is a Kubernetes-native registry of all bare metal and edge machines that have registered with your Rancher management cluster. Each registered machine appears as a MachineInventory resource containing hardware information, labels, and current adoption status. Effective inventory management ensures you can track, organize, and provision your entire edge fleet.

## Viewing Machine Inventory

```bash
# List all machines in inventory
kubectl get machineinventory -n fleet-default

# Get detailed output with labels
kubectl get machineinventory -n fleet-default \
  --show-labels \
  -o wide

# Get YAML for a specific machine
kubectl get machineinventory -n fleet-default <machine-name> -o yaml

# Describe a machine for full details
kubectl describe machineinventory -n fleet-default <machine-name>
```

## Understanding MachineInventory Fields

```yaml
# Example MachineInventory resource
apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventory
metadata:
  name: m-abc12345
  namespace: fleet-default
  labels:
    # Custom labels from MachineRegistration
    location: datacenter-1
    role: worker
    # Hardware labels collected via SMBIOS
    serialNumber: "SN1234567"
  annotations:
    # Registration timestamp
    elemental.cattle.io/registered: "2026-03-20T10:00:00Z"
spec:
  # TPM hash for secure identification
  tpmHash: "sha256:abc123..."
  # Reference to MachineRegistration used
  machineRegistrationRef:
    name: my-nodes
    namespace: fleet-default
  # When adopted, references the provisioned machine
  machineRef:
    name: m-abc12345
    namespace: fleet-default
```

## Labeling Machines

Labels are essential for organizing your inventory and targeting machines with selectors:

```bash
# Add a label to a machine
kubectl label machineinventory -n fleet-default m-abc12345 \
  rack=rack-01 \
  floor=1

# Update an existing label
kubectl label machineinventory -n fleet-default m-abc12345 \
  role=control-plane \
  --overwrite

# Remove a label
kubectl label machineinventory -n fleet-default m-abc12345 \
  temporary-

# Label multiple machines at once
kubectl label machineinventory -n fleet-default \
  -l location=datacenter-1 \
  status=ready
```

## Filtering and Querying Inventory

```bash
# Filter by label selector
kubectl get machineinventory -n fleet-default \
  -l location=datacenter-1

# Filter by multiple labels
kubectl get machineinventory -n fleet-default \
  -l "location=datacenter-1,role=worker"

# Find machines NOT yet adopted (available for provisioning)
kubectl get machineinventory -n fleet-default \
  -o json | jq '.items[] | select(.spec.machineRef == null) | .metadata.name'

# Find adopted machines
kubectl get machineinventory -n fleet-default \
  -o jsonpath='{range .items[?(@.spec.machineRef)]}{.metadata.name}{"\n"}{end}'
```

## Updating Machine Annotations

```bash
# Add a maintenance annotation
kubectl annotate machineinventory -n fleet-default m-abc12345 \
  maintenance.example.com/scheduled="2026-04-01T00:00:00Z" \
  maintenance.example.com/reason="firmware-update"

# Remove an annotation
kubectl annotate machineinventory -n fleet-default m-abc12345 \
  maintenance.example.com/scheduled-
```

## Exporting Inventory Data

```bash
# Export all inventory to JSON
kubectl get machineinventory -n fleet-default -o json > inventory.json

# Generate a CSV report of machines
kubectl get machineinventory -n fleet-default \
  -o custom-columns=\
"NAME:.metadata.name,\
LOCATION:.metadata.labels.location,\
ROLE:.metadata.labels.role,\
ADOPTED:.spec.machineRef.name" \
  --no-headers | tee inventory-report.csv

# Count machines by location
kubectl get machineinventory -n fleet-default \
  -o json | jq '[.items[].metadata.labels.location] | group_by(.) | map({location: .[0], count: length})'
```

## Deleting Machines from Inventory

```bash
# Remove a specific machine (will trigger re-provisioning if part of a cluster)
kubectl delete machineinventory -n fleet-default m-abc12345

# Remove all machines with a specific label
kubectl delete machineinventory -n fleet-default \
  -l status=decommissioned
```

## Conclusion

Managing the Elemental MachineInventory effectively is key to operating a large edge or bare metal fleet. By keeping inventory labels accurate and up to date, you enable precise targeting with MachineInventorySelectors for cluster provisioning. Regular inventory audits help identify unused machines, plan for capacity, and maintain an accurate picture of your infrastructure.
