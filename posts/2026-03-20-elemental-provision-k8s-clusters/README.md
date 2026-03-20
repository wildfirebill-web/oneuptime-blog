# How to Provision Kubernetes Clusters with Elemental

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Kubernetes, Rancher, Provisioning, Edge

Description: Learn how to use Elemental MachineInventorySelectors and Rancher cluster templates to provision Kubernetes clusters on registered bare metal nodes.

## Introduction

Once machines are registered in the Elemental MachineInventory, the next step is to provision them into Kubernetes clusters. Elemental integrates with Rancher's cluster provisioning to use MachineInventorySelectors that match registered machines to cluster roles (control plane, etcd, worker).

## Prerequisites

- Elemental Operator installed
- Machines registered in MachineInventory
- Rancher v2.7+ with cluster provisioning enabled

## Step 1: Verify Machines in Inventory

```bash
# List all registered machines

kubectl get machineinventory -n fleet-default

# Check machine labels
kubectl get machineinventory -n fleet-default --show-labels
```

## Step 2: Create MachineInventorySelector

The MachineInventorySelector defines which machines from inventory are eligible for a specific cluster role:

```yaml
# control-plane-selector.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventorySelector
metadata:
  name: control-plane-selector
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      role: control-plane
      location: datacenter-1
```

```bash
kubectl apply -f control-plane-selector.yaml
```

## Step 3: Create a Cluster Using Rancher UI

1. In Rancher, navigate to **Cluster Management** > **Create**
2. Select **Elemental** as the provisioning type
3. Configure the cluster name and Kubernetes version
4. Set up machine pools referencing your MachineInventorySelectors
5. Click **Create**

## Step 4: Create Cluster via YAML

```yaml
# elemental-cluster.yaml
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: my-edge-cluster
  namespace: fleet-default
spec:
  kubernetesVersion: v1.28.0+rke2r1
  rkeConfig:
    machinePools:
      # Control plane nodes
      - name: control-plane
        quantity: 3
        etcdRole: true
        controlPlaneRole: true
        workerRole: false
        machineConfigRef:
          kind: MachineInventorySelectorTemplate
          apiVersion: elemental.cattle.io/v1beta1
          name: cp-selector-template
      # Worker nodes
      - name: workers
        quantity: 5
        etcdRole: false
        controlPlaneRole: false
        workerRole: true
        machineConfigRef:
          kind: MachineInventorySelectorTemplate
          apiVersion: elemental.cattle.io/v1beta1
          name: worker-selector-template
```

## Step 5: Monitor Cluster Provisioning

```bash
# Watch the cluster status
kubectl get cluster -n fleet-default my-edge-cluster --watch

# Check machine adoption
kubectl get machineinventory -n fleet-default \
  -o custom-columns=NAME:.metadata.name,ADOPTED:.spec.machineRef.name

# View provisioning events
kubectl get events -n fleet-default --sort-by=.lastTimestamp
```

## Conclusion

Elemental's integration with Rancher's cluster provisioning creates a seamless pipeline from bare metal machine to production Kubernetes cluster. By using MachineInventorySelectors, you can declaratively define which machines form each cluster role, enabling reproducible and scalable edge Kubernetes deployments.
