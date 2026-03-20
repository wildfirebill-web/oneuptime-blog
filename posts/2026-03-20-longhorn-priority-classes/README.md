# How to Configure Longhorn Priority Classes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Priority Classes, Kubernetes, Storage, Resource Management, QoS, SUSE Rancher

Description: Learn how to configure Kubernetes PriorityClasses for Longhorn system components to protect storage operations from being preempted during node resource contention.

---

Longhorn system pods (manager, driver, CSI components) need to survive node resource pressure. Assigning appropriate PriorityClasses ensures Longhorn's components are protected from preemption and eviction.

---

## Why Priority Classes Matter for Longhorn

When a node runs low on memory or CPU:
- Kubernetes evicts the **lowest priority pods** first
- If Longhorn manager or engine pods are evicted, volumes become unavailable
- Setting a high priority class protects Longhorn from being evicted before user workloads

---

## Step 1: Create Priority Classes

Create a Longhorn-specific priority class:

```yaml
# longhorn-priority-class.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: longhorn-critical
# Value between kube-system components (2000000000) and user workloads (0)
value: 1000000
globalDefault: false
description: "Priority class for Longhorn storage components"
preemptionPolicy: PreemptLowerPriority
```

```bash
kubectl apply -f longhorn-priority-class.yaml
```

---

## Step 2: Configure Longhorn to Use the Priority Class

Set the priority class in Longhorn's global settings via the UI or directly:

```bash
# Update via kubectl
kubectl patch setting.longhorn.io priority-class \
  -n longhorn-system \
  --type merge \
  -p '{"value":"longhorn-critical"}'

# Verify the setting
kubectl get setting.longhorn.io priority-class \
  -n longhorn-system \
  -o jsonpath='{.value}'
```

After this change, Longhorn will restart its system pods with the new priority class applied.

---

## Step 3: Verify Priority Classes Are Applied

```bash
# Check that Longhorn pods have the priority class
kubectl get pods -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,PRIORITY:.spec.priorityClassName'
```

---

## Step 4: Configure Priority Class for Workloads

Define priority classes for your workloads relative to Longhorn:

```yaml
# storage-consumer-priority.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: storage-consumer-high
# Lower than longhorn-critical so Longhorn is protected
value: 500000
```

```yaml
# Reference in pod spec
spec:
  priorityClassName: storage-consumer-high
  containers:
    - name: app
      image: myapp:latest
```

---

## Priority Class Hierarchy for Longhorn Clusters

```
kube-system pods    ~2,000,000,000 (built-in)
longhorn-critical   ~1,000,000       (protect storage layer)
database-workloads  ~  800,000       (stateful apps)
api-services        ~  500,000       (stateless apps)
batch-jobs          ~  100,000       (low priority)
default (user pods) ~        0
```

---

## Best Practices

- Set Longhorn priority class **before** installing Longhorn, or restart Longhorn pods after updating.
- Do not set Longhorn's priority class above cluster system components — Longhorn should not preempt CoreDNS or kube-proxy.
- Create separate priority classes for your stateful vs. stateless workloads to control eviction order independently.
