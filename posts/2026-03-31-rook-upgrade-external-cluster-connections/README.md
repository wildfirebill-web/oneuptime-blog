# How to Upgrade External Cluster Connections in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, External Cluster, Upgrade, Connection

Description: Learn how to upgrade the connection between a Rook consumer cluster and an external Ceph provider cluster when the provider is updated or credentials change.

---

## Overview

When you run Rook in external cluster mode, the consumer Kubernetes cluster connects to an independently managed Ceph cluster. Over time, the provider cluster may be upgraded to a new Ceph version, monitor endpoints may change, or credentials may rotate. This guide explains how to update the external cluster connection in Rook without disrupting running workloads.

## When to Update the External Connection

You need to refresh the external connection when:

- The Ceph provider cluster is upgraded to a new major version
- Monitor IP addresses or hostnames change
- Client keyrings are rotated for security reasons
- New pools or CephFS filesystems are added to the provider

## Step 1 - Re-export Config from the Provider

Run the export script again on the provider cluster to generate fresh credentials and updated monitor endpoints:

```bash
python3 create-external-cluster-resources.py \
  --rbd-data-pool-name replicapool \
  --namespace rook-ceph-external \
  --format bash \
  > updated-external-config.sh
```

Review the diff between the old and new config:

```bash
diff external-cluster-config.sh updated-external-config.sh
```

## Step 2 - Update Kubernetes Secrets on the Consumer

Apply the updated secrets to the consumer cluster. The script will use `kubectl apply` which updates existing secrets in-place:

```bash
kubectl config use-context consumer-cluster
bash updated-external-config.sh
```

To manually update a specific secret with new monitor addresses:

```bash
kubectl -n rook-ceph-external create secret generic rook-ceph-mon \
  --from-literal=ceph-username=client.healthchecker \
  --from-literal=ceph-secret=<new-key> \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Step 3 - Update the ConfigMap with New Monitor Endpoints

If monitor IPs changed, update the ConfigMap that stores monitor addresses:

```bash
kubectl -n rook-ceph-external edit configmap rook-ceph-mon-endpoints
```

Or patch it directly:

```bash
kubectl -n rook-ceph-external patch configmap rook-ceph-mon-endpoints \
  --type merge \
  -p '{"data":{"data":"a=192.168.2.1:6789,b=192.168.2.2:6789,c=192.168.2.3:6789"}}'
```

## Step 4 - Restart the Rook Operator

After updating secrets and ConfigMaps, restart the Rook operator to pick up the new configuration:

```bash
kubectl -n rook-ceph rollout restart deployment/rook-ceph-operator
kubectl -n rook-ceph rollout status deployment/rook-ceph-operator
```

## Step 5 - Update the CephCluster Spec for New Ceph Version

If the provider Ceph version changed, update the CephCluster CRD's expected version so Rook health checks use the correct version comparison:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph-external
  namespace: rook-ceph-external
spec:
  external:
    enable: true
  cephVersion:
    image: quay.io/ceph/ceph:v18.2.0
```

```bash
kubectl apply -f external-cephcluster.yaml
```

## Step 6 - Verify Reconnection

Confirm the consumer cluster reconnects successfully:

```bash
kubectl -n rook-ceph-external get cephcluster
kubectl -n rook-ceph-external describe cephcluster rook-ceph-external | grep -i phase

# Check CSI driver connectivity
kubectl -n rook-ceph get pods | grep csi
```

## Summary

Upgrading external cluster connections in Rook involves re-exporting credentials from the provider, updating Kubernetes secrets and ConfigMaps in the consumer cluster, and restarting the Rook operator. Monitor endpoint changes require updating the `rook-ceph-mon-endpoints` ConfigMap explicitly. After any credential or endpoint update, always verify the CephCluster status returns to `Connected` before considering the upgrade complete.
