# How to Configure Elemental Machine Selectors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Machine Selectors, Kubernetes, Edge, CAPI

Description: Use MachineInventorySelector resources to match registered machines to cluster roles for automated Kubernetes provisioning.

## Introduction

MachineInventorySelectors are the bridge between the Elemental MachineInventory and Kubernetes cluster provisioning. They define label-based rules that match registered machines to specific cluster roles (control plane, etcd, worker). When the cluster provisioner needs a new node, it uses selectors to find available machines that meet the criteria.

## Understanding the Selection Process

1. Machine registers and gets labels from MachineRegistration
2. MachineInventorySelector defines label criteria
3. Cluster provisioning requests machines via selector
4. Operator matches available machines to requests
5. Machine is "adopted" and joins the cluster

## Creating a Basic MachineInventorySelector

```yaml
# basic-selector.yaml

apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventorySelector
metadata:
  name: worker-selector
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      # Match machines with these exact labels
      role: worker
      environment: production
```

```bash
kubectl apply -f basic-selector.yaml

# Verify the selector matches expected machines
kubectl get machineinventory -n fleet-default \
  -l role=worker,environment=production
```

## Advanced Label Selectors

```yaml
# advanced-selector.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventorySelector
metadata:
  name: gpu-worker-selector
  namespace: fleet-default
spec:
  selector:
    # matchLabels for exact matches
    matchLabels:
      role: worker
      environment: production

    # matchExpressions for advanced logic
    matchExpressions:
      # Must have a GPU
      - key: hardware.gpu
        operator: Exists

      # Must be in one of these datacenters
      - key: location
        operator: In
        values:
          - dc-east-1
          - dc-west-1

      # Must NOT be marked for maintenance
      - key: maintenance
        operator: DoesNotExist

      # Minimum memory (as label)
      - key: tier
        operator: NotIn
        values:
          - low-memory
```

## MachineInventorySelectorTemplate

For use with cluster provisioning, you need a template:

```yaml
# selector-template.yaml
apiVersion: elemental.cattle.io/v1beta1
kind: MachineInventorySelectorTemplate
metadata:
  name: worker-selector-template
  namespace: fleet-default
spec:
  template:
    spec:
      selector:
        matchLabels:
          role: worker
          ready: "true"
        matchExpressions:
          - key: location
            operator: In
            values:
              - datacenter-1
              - datacenter-2
```

```bash
kubectl apply -f selector-template.yaml

# Reference in cluster provisioning
kubectl get machineinventoryselectortemplate -n fleet-default
```

## Using Selectors in Cluster Provisioning

```yaml
# cluster-with-selectors.yaml
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: production-cluster
  namespace: fleet-default
spec:
  kubernetesVersion: v1.28.0+rke2r1
  rkeConfig:
    machinePools:
      - name: control-plane
        quantity: 3
        etcdRole: true
        controlPlaneRole: true
        workerRole: false
        machineConfigRef:
          kind: MachineInventorySelectorTemplate
          apiVersion: elemental.cattle.io/v1beta1
          name: cp-selector-template

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

## Checking Selector Matches

```bash
# Find how many machines match a selector
kubectl get machineinventory -n fleet-default \
  -l role=worker,environment=production | wc -l

# Find available (non-adopted) machines matching selector
kubectl get machineinventory -n fleet-default \
  -l role=worker,environment=production \
  -o json | jq '[.items[] | select(.spec.machineRef == null)] | length'

# Show all selector resources
kubectl get machineinventoryselector -n fleet-default
kubectl get machineinventoryselectortemplate -n fleet-default
```

## Conclusion

MachineInventorySelectors provide flexible, label-based matching between your physical machine inventory and Kubernetes cluster roles. By designing a clear labeling taxonomy for your machines and creating selectors that match your infrastructure topology, you enable fully automated, declarative cluster provisioning that scales from small deployments to large fleets.
