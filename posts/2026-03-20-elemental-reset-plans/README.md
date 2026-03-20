# How to Configure Elemental Reset Plans

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Reset, Kubernetes, Edge, Operations

Description: Learn how to use Elemental reset plans to wipe and re-provision nodes back to a clean state declaratively.

## Introduction

Elemental reset plans allow you to wipe and re-provision nodes back to their original state without manual intervention. This is useful for decommissioning nodes, recovering from corruption, or repurposing machines from one cluster to another. The reset process is triggered declaratively through a Kubernetes resource, making it auditable and repeatable.

## Understanding the Reset Process

When a reset is triggered:
1. The node receives the reset signal
2. On next reboot, the Elemental reset process runs
3. Persistent data is optionally wiped
4. The node reinstalls from the original OS image
5. The node re-registers with the MachineRegistration

## Configuring Reset in Cloud-Config

Include reset configuration in your MachineRegistration:

```yaml
# machine-registration-with-reset.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: MachineRegistration
metadata:
  name: my-nodes
  namespace: fleet-default
spec:
  config:
    elemental:
      reset:
        # Reboot after reset completes
        reboot: true
        # Wipe the persistent partition
        reset-persistent: true
        # Wipe the OEM partition (registration info)
        reset-oem: false
        # Additional cloud-config applied after reset
        cloud-config:
          users:
            - name: root
              passwd: "$6$rounds=4096$salt$hashedpass"
```

## Triggering a Reset via MachineInventory

```bash
# Annotate a machine to trigger reset
kubectl annotate machineinventory -n fleet-default m-abc12345 \
  elemental.cattle.io/os.unmanaged="true"

# Or patch the machine inventory
kubectl patch machineinventory -n fleet-default m-abc12345 \
  --type merge \
  -p '{"metadata":{"annotations":{"elemental.cattle.io/reset":"true"}}}'
```

## Creating a Reset Plan Resource

```yaml
# reset-plan.yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: elemental-reset
  namespace: cattle-system
spec:
  # Channel or image for reset
  channel: "https://my-registry.example.com/reset-channel:latest"

  # Node selector for which nodes to reset
  nodeSelector:
    matchLabels:
      elemental.cattle.io/reset: "true"

  # Tolerations for tainted nodes
  tolerations:
    - key: "node.kubernetes.io/unschedulable"
      operator: "Exists"
      effect: "NoSchedule"

  # Upgrade/reset container configuration
  upgrade:
    image: registry.suse.com/rancher/elemental-toolkit/elemental-cli
    command:
      - elemental
      - reset
      - --reset-persistent
      - --reboot
```

## Resetting Specific Nodes

```bash
# Label a node for reset
kubectl label node worker-01 elemental.cattle.io/reset="true"

# Apply the reset plan
kubectl apply -f reset-plan.yaml

# Watch the reset progress
kubectl get pods -n cattle-system \
  -l upgrade.cattle.io/plan=elemental-reset --watch
```

## Reset with Data Preservation

```yaml
# Partial reset - preserve specific persistent data
apiVersion: elemental.cattle.io/v1beta1
kind: MachineRegistration
metadata:
  name: preserve-data-nodes
  namespace: fleet-default
spec:
  config:
    elemental:
      reset:
        reboot: true
        # Keep persistent partition (app data)
        reset-persistent: false
        # Wipe OEM partition (registration info)
        reset-oem: true
        cloud-config:
          runcmd:
            # Custom cleanup script
            - /usr/local/bin/cleanup-app-data.sh
```

## Monitoring Reset Operations

```bash
# Check which machines are in reset state
kubectl get machineinventory -n fleet-default \
  -o json | jq '.items[] | select(.metadata.annotations["elemental.cattle.io/reset"] == "true") | .metadata.name'

# Watch for machines re-registering after reset
kubectl get machineinventory -n fleet-default --watch

# Check logs for reset operations
kubectl logs -n elemental-system -l app=elemental-operator \
  --since=1h | grep -i reset
```

## Bulk Reset Operations

```bash
# Reset all machines in a specific location
kubectl label machineinventory -n fleet-default \
  -l location=datacenter-old \
  elemental.cattle.io/reset="true"

# Apply the reset plan targeting labeled nodes
kubectl apply -f reset-plan.yaml
```

## Conclusion

Elemental reset plans provide a clean, auditable way to wipe and re-provision nodes without manual intervention. Whether you're decommissioning hardware, recovering from OS corruption, or repurposing nodes for different workloads, the declarative reset mechanism ensures the process is consistent and traceable. After reset, nodes automatically re-register and become available for new cluster assignments.
