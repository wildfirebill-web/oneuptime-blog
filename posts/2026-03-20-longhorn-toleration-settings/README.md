# How to Configure Longhorn Toleration Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Tolerations, Taints, Kubernetes, Storage, Node Selection, SUSE Rancher

Description: Learn how to configure Longhorn global toleration settings so Longhorn system pods can be scheduled on tainted nodes, enabling dedicated storage nodes in your Kubernetes cluster.

---

Kubernetes node taints prevent non-tolerating pods from being scheduled on nodes. When you reserve dedicated storage nodes with custom taints, Longhorn's system pods need matching tolerations to run on those nodes.

---

## Use Case: Dedicated Storage Nodes

A common pattern is to add a `dedicated=storage:NoSchedule` taint to specific nodes reserved for Longhorn, and configure Longhorn to tolerate this taint so only Longhorn pods run on those nodes.

---

## Step 1: Taint Storage-Dedicated Nodes

```bash
# Taint nodes reserved for Longhorn storage
kubectl taint node storage-node-01 dedicated=storage:NoSchedule
kubectl taint node storage-node-02 dedicated=storage:NoSchedule
kubectl taint node storage-node-03 dedicated=storage:NoSchedule

# Verify taints
kubectl describe node storage-node-01 | grep Taints
```

---

## Step 2: Configure Longhorn Tolerations

Set the toleration in Longhorn's global settings. This applies to the Longhorn manager, engine, CSI, and UI pods:

```bash
# Set via kubectl
kubectl patch setting.longhorn.io taint-toleration \
  -n longhorn-system \
  --type merge \
  -p '{"value":"dedicated=storage:NoSchedule"}'
```

For multiple tolerations, separate with semicolons:

```bash
# Multiple tolerations
kubectl patch setting.longhorn.io taint-toleration \
  -n longhorn-system \
  --type merge \
  -p '{"value":"dedicated=storage:NoSchedule;node-role=longhorn:NoExecute"}'
```

---

## Step 3: Restart Longhorn Pods to Apply Tolerations

After updating the toleration setting, Longhorn will restart its pods to apply the change:

```bash
# Watch Longhorn pods restart with the new toleration
kubectl get pods -n longhorn-system -w

# Verify tolerations are applied
kubectl get pods -n longhorn-system -o json | \
  jq '.items[].spec.tolerations'
```

---

## Step 4: Restrict Longhorn to Storage Nodes Only

To ensure Longhorn only runs on storage nodes, combine tolerations with a node selector:

```bash
# Set node selector to run Longhorn only on labeled storage nodes
kubectl label node storage-node-01 node-type=longhorn
kubectl label node storage-node-02 node-type=longhorn

# Set the Longhorn system managed component node selector
kubectl patch setting.longhorn.io system-managed-components-node-selector \
  -n longhorn-system \
  --type merge \
  -p '{"value":"node-type:longhorn"}'
```

---

## Step 5: Test Toleration Is Working

Deploy a test pod without the toleration — it should not be scheduled on tainted nodes:

```bash
# This pod should not be scheduled on tainted storage nodes
kubectl run no-toleration-pod --image=nginx

# Check which node it was scheduled on
kubectl get pod no-toleration-pod -o jsonpath='{.spec.nodeName}'
```

---

## Best Practices

- Use `NoSchedule` taints (not `NoExecute`) for storage-only nodes unless you want to forcibly evict existing pods.
- After adding tolerations, verify that Longhorn replicas are being placed on the storage-only nodes as intended.
- Document your taint and toleration scheme — it is easy to accidentally add new nodes that are not tainted, leading to uneven storage distribution.
