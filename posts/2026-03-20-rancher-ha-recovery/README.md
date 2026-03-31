# How to Recover from Rancher HA Node Failure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, High Availability, Recovery, Disaster Recovery, etcd

Description: Step-by-step guide to recovering Rancher HA from node failures, etcd member loss, and worst-case scenarios including complete cluster restoration from backup.

## Introduction

When a Rancher HA node fails, the recovery procedure depends on the type of failure. This guide covers recovery for single node failures, etcd member removal and replacement, and restoration from etcd snapshots when the cluster cannot recover automatically.

## Prerequisites

- Access to remaining healthy cluster nodes
- etcd snapshots configured (automatic or manual)
- SSH access to all cluster nodes
- Documented node configuration

## Step 1: Assess the Failure

```bash
# Determine the failure type

kubectl get nodes
kubectl get pods -n kube-system | grep etcd
kubectl get pods -n cattle-system

# Check etcd status (if API server is accessible)
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key \
  -w table

# Check RKE2 service status on each node
for node in rke2-server-01 rke2-server-02 rke2-server-03; do
  echo "=== $node ==="
  ssh $node "systemctl status rke2-server | head -5"
done
```

## Step 2: Single Node Recovery (Node Still Accessible)

```bash
# If the node failed but is still reachable

# Check what's wrong on the failed node
ssh rke2-server-02 "journalctl -u rke2-server --since='1 hour ago' | tail -50"

# Check disk space (common cause of failure)
ssh rke2-server-02 "df -h"

# Check memory
ssh rke2-server-02 "free -h"

# Restart RKE2 service
ssh rke2-server-02 "systemctl restart rke2-server"

# Wait for node to rejoin
kubectl get nodes -w

# Verify etcd member rejoined
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key
```

## Step 3: Replace Failed Node

```bash
# If the node cannot be recovered, replace it

# Step 1: Remove failed etcd member from cluster
# Get member ID of failed node
FAILED_MEMBER_ID=$(kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key \
  | grep "rke2-server-02" | awk -F, '{print $1}')

kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl member remove $FAILED_MEMBER_ID \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key

# Step 2: Delete the failed node from Kubernetes
kubectl delete node rke2-server-02

# Step 3: Provision a new replacement node
# Install RKE2 on new node with same configuration
cat > /etc/rancher/rke2/config.yaml << EOF
token: "my-cluster-token"
server: https://rke2-server-01:9345
tls-san:
  - rancher.example.com
  - 10.0.0.100
EOF

curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server
systemctl start rke2-server

# Verify new node joined
kubectl get nodes -w
```

## Step 4: Restore from etcd Snapshot

```bash
# LAST RESORT: If cluster cannot recover automatically
# Only when etcd quorum is lost (e.g., 2 of 3 nodes down)

# List available snapshots on the surviving node
ls /var/lib/rancher/rke2/server/db/snapshots/

# Or for RKE2
rke2 etcd-snapshot list

# Stop RKE2 on ALL remaining nodes
systemctl stop rke2-server

# Restore from snapshot on the first server
rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/etcd-snapshot-date

# Wait for reset to complete, then start RKE2
systemctl start rke2-server

# After first node is running, join others
# On other nodes: (they will sync from the restored snapshot)
systemctl start rke2-server
```

## Step 5: Recover Rancher Pods

```bash
# After cluster recovery, check Rancher pods
kubectl get pods -n cattle-system

# If Rancher pods are not recovering
kubectl describe pod -n cattle-system \
  $(kubectl get pod -n cattle-system -l app=rancher -o name | head -1)

# Force restart Rancher deployment
kubectl rollout restart deployment/rancher -n cattle-system

# Watch rollout
kubectl rollout status deployment/rancher -n cattle-system

# Verify Rancher is accessible
curl -sk https://rancher.example.com/ping
```

## Step 6: Verify Managed Cluster Reconnection

```bash
# After Rancher recovery, managed clusters should reconnect automatically
kubectl get clusters.management.cattle.io

# If a cluster shows disconnected after 10+ minutes
# Check cattle-cluster-agent in the managed cluster
kubectl get pods -n cattle-system --context=managed-cluster

# The agent will retry connection automatically
# You can force reconnection by restarting the agent
kubectl rollout restart deployment/cattle-cluster-agent \
  -n cattle-system \
  --context=managed-cluster
```

## Step 7: Post-Recovery Validation

```bash
# Run full validation after recovery

# 1. etcd cluster health
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl endpoint health \
  --cluster \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key

# 2. All Rancher pods running
kubectl get pods -n cattle-system

# 3. Rancher accessible
curl -sk https://rancher.example.com/ping

# 4. Managed clusters connected
kubectl get clusters.management.cattle.io

# 5. Fleet sync status
kubectl get gitrepos -n fleet-default

# 6. Take fresh etcd snapshot
rke2 etcd-snapshot save \
  --name post-recovery-$(date +%Y%m%d)
```

## Conclusion

Rancher HA recovery is straightforward for single-node failures-the cluster recovers automatically. The procedure becomes more involved for hardware replacement, requiring manual etcd member removal and node re-provisioning. The most extreme case-restoring from etcd snapshot-should be well-practiced through regular disaster recovery drills. The key prerequisites for fast recovery are: automated etcd snapshots to S3 or cloud storage, documented cluster token and configuration, and rehearsed procedures that your team can execute under pressure.
