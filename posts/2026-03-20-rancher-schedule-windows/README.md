# How to Schedule Workloads on Windows Nodes in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Windows, Scheduling, Node Selectors, Affinity

Description: Configure Kubernetes scheduling rules to direct Windows workloads to Windows nodes using node selectors, node affinity, taints, and tolerations in Rancher clusters.

## Introduction

Proper workload scheduling in mixed OS clusters ensures that Windows containers always run on Windows nodes and Linux containers on Linux nodes. Incorrect scheduling results in pod failures with "image OS mismatch" errors. This guide covers all scheduling mechanisms for directing workloads to the correct nodes in Rancher.

## Prerequisites

- Rancher cluster with both Linux and Windows worker nodes
- Windows nodes labeled with `kubernetes.io/os=windows`
- kubectl access

## Step 1: Basic Node Selector

```yaml
# Simplest approach: nodeSelector

# Every Windows workload MUST have this
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-app
  namespace: production
spec:
  template:
    spec:
      # This is mandatory for Windows pods
      nodeSelector:
        kubernetes.io/os: windows
      containers:
        - name: app
          image: registry.example.com/windows-app:v1.0

# For Linux workloads, explicitly target Linux nodes
# (prevents accidental scheduling on Windows if all Linux nodes are full)
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux
```

## Step 2: Node Affinity for Advanced Scheduling

```yaml
# Use node affinity for more complex scheduling rules
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-framework-app
  namespace: production
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          # Required: must run on Windows
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - windows
                  # Require Windows Server 2022 (not 2019)
                  - key: node.kubernetes.io/windows-build
                    operator: In
                    values:
                      - "10.0.20348"  # Windows Server 2022
          # Preferred: prefer nodes with SSD
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: storage-type
                    operator: In
                    values:
                      - ssd
      containers:
        - name: app
          image: registry.example.com/dotnet-app:v1.0
```

## Step 3: Taints and Tolerations

```bash
# Taint Windows nodes to prevent Linux workloads from scheduling there
kubectl taint nodes win-node-01 os=windows:NoSchedule
kubectl taint nodes win-node-02 os=windows:NoSchedule
kubectl taint nodes win-node-03 os=windows:NoSchedule

# Now all Linux workloads will NOT schedule on Windows nodes
# Windows workloads must add toleration:
```

```yaml
# Windows deployment with toleration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-app
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      # Required to schedule on tainted Windows nodes
      tolerations:
        - key: os
          operator: Equal
          value: windows
          effect: NoSchedule
      containers:
        - name: app
          image: registry.example.com/windows-app:v1.0
```

## Step 4: Namespace-Level Default Scheduling

```yaml
# Use LimitRanger or admission webhook to add nodeSelector to all pods in a namespace

# Alternatively, use Kyverno to inject nodeSelector
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: windows-namespace-policy
spec:
  rules:
    - name: add-windows-node-selector
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaceSelector:
                matchLabels:
                  workload-os: windows  # Label the namespace
      mutate:
        patchStrategicMerge:
          spec:
            nodeSelector:
              kubernetes.io/os: windows
            tolerations:
              - key: os
                operator: Equal
                value: windows
                effect: NoSchedule
```

```bash
# Label namespace for automatic Windows scheduling
kubectl label namespace windows-apps workload-os=windows
```

## Step 5: Pod Anti-Affinity for High Availability

```yaml
# Spread Windows pods across multiple Windows nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-ha-app
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          value: windows
          effect: NoSchedule
      affinity:
        podAntiAffinity:
          # Required: spread across different Windows nodes
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - windows-ha-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: app
          image: registry.example.com/windows-app:v1.0
```

## Step 6: Topology Spread Constraints

```yaml
# Advanced: spread Windows pods evenly across Windows nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-spread-app
  namespace: production
spec:
  replicas: 6
  template:
    metadata:
      labels:
        app: windows-spread-app
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: os
          value: windows
          effect: NoSchedule
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: windows-spread-app
      containers:
        - name: app
          image: registry.example.com/windows-app:v1.0
```

## Step 7: Validate Scheduling Configuration

```bash
# Verify Windows pods are scheduled on Windows nodes
kubectl get pods -n production -l app=windows-app \
  -o custom-columns="NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase"

# Check node OS type
kubectl get nodes -o custom-columns="NAME:.metadata.name,OS:.status.nodeInfo.operatingSystem,ARCH:.status.nodeInfo.architecture"

# Verify Windows nodes have correct labels
kubectl get nodes -l kubernetes.io/os=windows --show-labels

# Test scheduling: create a Windows pod and verify it lands on a Windows node
kubectl run test-windows-pod \
  --image=mcr.microsoft.com/windows/nanoserver:ltsc2022 \
  --overrides='{"spec":{"nodeSelector":{"kubernetes.io/os":"windows"},"tolerations":[{"key":"os","value":"windows","effect":"NoSchedule"}]}}' \
  -- cmd.exe /c "echo hello && timeout /t 30"

kubectl get pod test-windows-pod -o wide
kubectl delete pod test-windows-pod
```

## Step 8: Scheduling Priority for Windows Workloads

```yaml
# Create PriorityClass for critical Windows workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: windows-critical
value: 1000
globalDefault: false
description: "Critical Windows workloads"
---
# Apply to deployment
spec:
  template:
    spec:
      priorityClassName: windows-critical
      nodeSelector:
        kubernetes.io/os: windows
```

## Conclusion

Proper workload scheduling is the foundation of reliable mixed OS clusters in Rancher. The minimum requirement is always specifying `kubernetes.io/os: windows` in nodeSelector for Windows pods. Tainting Windows nodes with `os=windows:NoSchedule` provides a safety net against accidental cross-OS scheduling. For high-availability Windows deployments, combine pod anti-affinity or topology spread constraints to ensure pods distribute across multiple Windows nodes. Namespace-level policies via Kyverno can automatically apply Windows scheduling rules, reducing the risk of misconfigured pods reaching Windows nodes.
