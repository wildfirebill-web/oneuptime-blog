# How to Sequence Helm Chart Upgrades (rook-ceph Before rook-ceph-cluster)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Helm, Upgrade, Sequencing, Kubernetes

Description: Learn the correct upgrade sequence for Rook Helm charts - upgrading rook-ceph operator before rook-ceph-cluster to avoid version mismatches.

---

Rook deployments using Helm typically involve two separate charts: `rook-ceph` (the operator) and `rook-ceph-cluster` (the cluster configuration). These must be upgraded in a specific order to avoid issues where the cluster chart references features or APIs not yet available in the running operator.

## Understanding the Two-Chart Architecture

```text
rook-ceph chart:
  - Deploys the Rook operator
  - Installs CRDs (Custom Resource Definitions)
  - Creates RBAC rules and service accounts
  - Manages the operator Deployment

rook-ceph-cluster chart:
  - Deploys CephCluster CR
  - Deploys CephBlockPool, CephFilesystem, etc.
  - Creates StorageClasses
  - Manages Ceph daemon configuration
```

The `rook-ceph-cluster` chart creates resources that are reconciled by the `rook-ceph` operator. If the operator is older than the cluster chart expects, the operator may not understand new CRD fields or features.

## Why Order Matters

Upgrading `rook-ceph-cluster` before `rook-ceph` can cause:
- New CRD fields being ignored (operator does not recognize them)
- Validation failures if new fields are required
- Incorrect behavior from spec drift between operator version and CR version
- In rare cases, data access interruption from misconfiguration

Always upgrade the operator chart first.

## Step 1: Upgrade rook-ceph (Operator)

Update the Helm repository:

```bash
helm repo update
```

Check what is currently installed:

```bash
helm list -n rook-ceph
```

```text
NAME                    NAMESPACE   REVISION  UPDATED   STATUS    CHART                    APP VERSION
rook-ceph               rook-ceph   3         2024-01   deployed  rook-ceph-1.14.0         1.14.0
rook-ceph-cluster       rook-ceph   5         2024-01   deployed  rook-ceph-cluster-1.14.0 1.14.0
```

Upgrade the operator chart first:

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --version 1.15.0 \
  --reuse-values
```

Wait for the operator to become ready:

```bash
kubectl -n rook-ceph rollout status deployment rook-ceph-operator
```

```text
deployment "rook-ceph-operator" successfully rolled out
```

## Step 2: Verify Operator is Running New Version

Confirm the operator is running the target version before proceeding:

```bash
kubectl -n rook-ceph get deployment rook-ceph-operator \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

```text
rook/ceph:v1.15.0
```

Check operator logs for any errors after the update:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator --tail=50
```

## Step 3: Verify Cluster Health After Operator Upgrade

Before touching the cluster chart, verify the cluster is still healthy:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

The operator upgrade should not change any Ceph daemon behavior. If the cluster shows any new errors, investigate before continuing.

## Step 4: Upgrade rook-ceph-cluster

With the operator running the new version, upgrade the cluster chart:

```bash
helm upgrade rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  --version 1.15.0 \
  --reuse-values
```

If you need to change values during the upgrade, pass them explicitly:

```bash
helm upgrade rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  --version 1.15.0 \
  --set cephClusterSpec.cephVersion.image=quay.io/ceph/ceph:v19.2.0 \
  --reuse-values
```

## Step 5: Monitor Cluster Reconciliation

The cluster chart upgrade will update the CephCluster CR, which triggers the operator to reconcile. Watch for changes:

```bash
kubectl -n rook-ceph get cephcluster -w
```

Monitor for any issues:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator -f | grep -E "reconcil|error|warn"
```

## Handling CRD Upgrades Between Minor Versions

For minor version upgrades (e.g., v1.14 to v1.15), CRDs may change. The `rook-ceph` chart can manage CRDs directly, but Helm has limitations with CRD lifecycle management. Verify CRD status:

```bash
kubectl get crd | grep rook
```

If using `--reuse-values`, ensure your existing values are compatible with the new chart version:

```bash
helm diff upgrade rook-ceph-cluster rook-release/rook-ceph-cluster \
  --version 1.15.0 \
  --reuse-values
```

## Automating Sequenced Upgrades

Create a script to enforce correct upgrade order:

```bash
#!/bin/bash
set -euo pipefail

TARGET_VERSION="${1}"
NAMESPACE="rook-ceph"

echo "Upgrading rook-ceph operator to ${TARGET_VERSION}"
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace "${NAMESPACE}" \
  --version "${TARGET_VERSION}" \
  --reuse-values --wait

echo "Operator upgraded. Checking cluster health..."
kubectl -n "${NAMESPACE}" exec deploy/rook-ceph-tools -- ceph status

echo "Upgrading rook-ceph-cluster to ${TARGET_VERSION}"
helm upgrade rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace "${NAMESPACE}" \
  --version "${TARGET_VERSION}" \
  --reuse-values --wait

echo "Upgrade complete."
```

## Summary

Helm upgrades for Rook must follow the order: `rook-ceph` operator first, then `rook-ceph-cluster`. The operator must be running the target version before the cluster chart is updated, so new CRD fields and features are recognized. Use `--wait` flags and intermediate health checks between the two upgrades to catch any issues early and prevent the cluster chart from running against an incompatible operator version.
