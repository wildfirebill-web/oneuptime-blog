# How to Set Up Storage Quotas in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Storage, Persistent Volume

Description: Learn how to configure storage quotas in Rancher to control and limit storage consumption across namespaces and teams.

Storage quotas prevent any single namespace or team from consuming all available cluster storage. Kubernetes provides ResourceQuotas and LimitRanges for controlling storage consumption. Rancher adds project-level quotas for broader management. This guide covers all approaches.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- A managed Kubernetes cluster with dynamic provisioning configured
- kubectl access to your cluster

## Step 1: Create a Namespace-Level Storage Quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-alpha
spec:
  hard:
    requests.storage: 500Gi
    persistentvolumeclaims: "20"
```

```bash
kubectl create namespace team-alpha
kubectl apply -f storage-quota.yaml
```

This limits the `team-alpha` namespace to 500Gi total storage and 20 PVCs.

## Step 2: Set Per-StorageClass Quotas

Limit how much of each storage tier a namespace can use:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tiered-storage-quota
  namespace: team-alpha
spec:
  hard:
    # Total storage limit
    requests.storage: 500Gi
    persistentvolumeclaims: "20"
    # Per-class limits
    premium.storageclass.storage.k8s.io/requests.storage: 100Gi
    premium.storageclass.storage.k8s.io/persistentvolumeclaims: "5"
    standard.storageclass.storage.k8s.io/requests.storage: 300Gi
    standard.storageclass.storage.k8s.io/persistentvolumeclaims: "15"
    economy.storageclass.storage.k8s.io/requests.storage: 100Gi
    economy.storageclass.storage.k8s.io/persistentvolumeclaims: "10"
```

```bash
kubectl apply -f tiered-quota.yaml
```

## Step 3: Configure Storage Quotas via Rancher UI

1. Navigate to your cluster in Rancher.
2. Go to **Cluster** > **Projects/Namespaces**.
3. Select or create a project.
4. Click **Edit** on the project.
5. Under **Resource Quotas**, configure:
   - **Storage Requests**: Set the limit (e.g., 500Gi)
   - **Persistent Volume Claims**: Set the maximum count
6. Click **Save**.

Rancher project quotas are distributed across namespaces within the project.

## Step 4: Set Default Storage Limits with LimitRange

Enforce default PVC sizes:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limits
  namespace: team-alpha
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 50Gi
    min:
      storage: 1Gi
    default:
      storage: 10Gi
    defaultRequest:
      storage: 5Gi
```

```bash
kubectl apply -f storage-limits.yaml
```

This ensures:
- No PVC can request more than 50Gi
- No PVC can request less than 1Gi
- PVCs without a size default to 10Gi

## Step 5: Configure Project-Level Quotas in Rancher

Rancher projects allow you to set quotas that apply across multiple namespaces:

1. Go to **Cluster Management** > select your cluster.
2. Go to **Projects/Namespaces**.
3. Click **Create Project** or edit an existing one.
4. Under **Resource Quotas**, add:
   - **Project Limit**: Total storage for the entire project
   - **Namespace Default Limit**: Default per-namespace allocation
5. Click **Create** or **Save**.

Example project quota configuration:
- Project limit: 1000Gi total storage, 50 PVCs
- Namespace default: 200Gi storage, 10 PVCs

Each namespace in the project gets up to 200Gi unless overridden.

## Step 6: Monitor Storage Quota Usage

```bash
# Check quota usage for a namespace

kubectl describe resourcequota storage-quota -n team-alpha

# Output example:
# Name:                                                     storage-quota
# Namespace:                                                team-alpha
# Resource                                                  Used    Hard
# --------                                                  ----    ----
# persistentvolumeclaims                                    5       20
# premium.storageclass.storage.k8s.io/persistentvolumeclaims  2    5
# premium.storageclass.storage.k8s.io/requests.storage     40Gi    100Gi
# requests.storage                                          150Gi   500Gi

# Check all quotas across namespaces
kubectl get resourcequota --all-namespaces
```

## Step 7: Set Up Quota Alerts

Create a monitoring alert for storage quota usage:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-quota-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: storage-quotas
    rules:
    - alert: StorageQuotaNearLimit
      expr: |
        kube_resourcequota{resource="requests.storage", type="used"} /
        kube_resourcequota{resource="requests.storage", type="hard"} > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Storage quota usage above 80% in namespace {{ $labels.namespace }}"
    - alert: PVCQuotaNearLimit
      expr: |
        kube_resourcequota{resource="persistentvolumeclaims", type="used"} /
        kube_resourcequota{resource="persistentvolumeclaims", type="hard"} > 0.9
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "PVC count quota usage above 90% in namespace {{ $labels.namespace }}"
```

## Step 8: Enforce Storage Policies with OPA/Gatekeeper

Create policies that enforce storage rules beyond what quotas provide:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8smaxpvcsize
spec:
  crd:
    spec:
      names:
        kind: K8sMaxPVCSize
      validation:
        openAPIV3Schema:
          type: object
          properties:
            maxSize:
              type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8smaxpvcsize
      violation[{"msg": msg}] {
        input.review.object.kind == "PersistentVolumeClaim"
        requested := input.review.object.spec.resources.requests.storage
        max := input.parameters.maxSize
        msg := sprintf("PVC size %v exceeds maximum allowed size %v", [requested, max])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMaxPVCSize
metadata:
  name: max-pvc-100gi
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["PersistentVolumeClaim"]
    namespaces: ["team-alpha", "team-beta"]
  parameters:
    maxSize: "100Gi"
```

## Step 9: Handle Quota Exceeded Errors

When a PVC creation fails due to quota:

```bash
# Check the error
kubectl describe pvc new-pvc -n team-alpha

# Error: exceeded quota: storage-quota, requested: requests.storage=100Gi,
# used: requests.storage=450Gi, limited: requests.storage=500Gi

# Review current usage
kubectl describe resourcequota storage-quota -n team-alpha

# Options:
# 1. Delete unused PVCs
kubectl get pvc -n team-alpha
kubectl delete pvc unused-pvc -n team-alpha

# 2. Request a quota increase (edit the quota)
kubectl edit resourcequota storage-quota -n team-alpha

# 3. Use a smaller PVC size
```

## Step 10: Create a Storage Quota Report

Generate a report of storage usage across all namespaces:

```bash
#!/bin/bash
echo "Namespace | Used Storage | Quota Limit | PVCs Used | PVC Limit"
echo "----------|-------------|-------------|-----------|----------"

for ns in $(kubectl get resourcequota --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\n"}{end}' | sort -u); do
  USED=$(kubectl get resourcequota -n $ns -o jsonpath='{.items[0].status.used.requests\.storage}' 2>/dev/null)
  HARD=$(kubectl get resourcequota -n $ns -o jsonpath='{.items[0].status.hard.requests\.storage}' 2>/dev/null)
  PVC_USED=$(kubectl get resourcequota -n $ns -o jsonpath='{.items[0].status.used.persistentvolumeclaims}' 2>/dev/null)
  PVC_HARD=$(kubectl get resourcequota -n $ns -o jsonpath='{.items[0].status.hard.persistentvolumeclaims}' 2>/dev/null)
  echo "$ns | ${USED:-N/A} | ${HARD:-N/A} | ${PVC_USED:-N/A} | ${PVC_HARD:-N/A}"
done
```

## Troubleshooting

- **Quota not enforced**: Ensure the ResourceQuota is in the correct namespace
- **Cannot create PVC**: Check quota usage with `kubectl describe resourcequota`
- **Per-class quota not working**: Verify the StorageClass name matches exactly in the quota spec
- **Project quota not applying**: Check Rancher project configuration and namespace membership
- **LimitRange conflicts**: Ensure min/max/default values are consistent

## Summary

Storage quotas in Rancher provide essential governance for multi-tenant Kubernetes environments. By combining namespace-level ResourceQuotas, per-StorageClass limits, LimitRanges for default sizing, and Rancher project-level quotas, you can control storage consumption effectively. Monitoring quota usage and setting up alerts ensures you catch capacity issues before they impact your teams.
