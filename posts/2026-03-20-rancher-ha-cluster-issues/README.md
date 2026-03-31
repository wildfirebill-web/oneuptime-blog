# How to Troubleshoot Rancher HA Cluster Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, has, High Availability

Description: Diagnose and resolve Rancher high-availability cluster issues, including split-brain scenarios, leader election failures, and etcd quorum problems.

## Introduction

Running Rancher in high-availability mode (3+ replicas backed by an HA Kubernetes cluster) introduces distributed systems challenges: leader election, database consistency, and replica synchronization. When things go wrong in an HA setup, the impact is cluster-wide. This guide provides a systematic approach to troubleshooting Rancher HA issues.

## Rancher HA Architecture

```text
Load Balancer (L4 or L7)
    ↓
[Rancher Pod 1] [Rancher Pod 2] [Rancher Pod 3]  ← cattle-system namespace
    ↓                ↓               ↓
         External MySQL Database (HA)
              (or embedded etcd)
```

## Step 1: Check All Rancher Replicas

```bash
# Verify all replicas are running and ready

kubectl get deployment -n cattle-system rancher
kubectl get pods -n cattle-system -l app=rancher -o wide

# Check replica health - all should show 1/1 Ready
# NAME                     READY   STATUS    RESTARTS   NODE
# rancher-7d9f6b8c9-aaa   1/1     Running   0          node1
# rancher-7d9f6b8c9-bbb   1/1     Running   0          node2
# rancher-7d9f6b8c9-ccc   1/1     Running   0          node3

# Get logs from ALL replicas
for pod in $(kubectl get pods -n cattle-system -l app=rancher -o name); do
  echo "=== Logs from ${pod} ==="
  kubectl logs -n cattle-system "${pod}" --tail=50 | tail -10
done
```

## Step 2: Check the Underlying HA Kubernetes Cluster

Rancher itself runs on a Kubernetes cluster (often RKE2 or K3s HA):

```bash
# Check all control plane nodes
kubectl get nodes -l node-role.kubernetes.io/control-plane=true

# Check etcd health (for RKE2/K3s HA)
# On each control-plane node:
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  endpoint health

# Expected output:
# https://127.0.0.1:2379 is healthy: successfully committed proposal

# Check etcd member list
sudo /var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/server-client.key \
  member list
```

## Step 3: Troubleshoot Leader Election

Rancher pods elect a leader for certain operations. If leader election is failing:

```bash
# Check Kubernetes lease resources used for leader election
kubectl get lease -n cattle-system

# Check for errors in the leader election logs
kubectl logs -n cattle-system -l app=rancher --tail=500 \
  | grep -iE "leader|election|lease"

# If a single replica holds the lease too long, force a rollout
kubectl rollout restart deployment/rancher -n cattle-system
kubectl rollout status deployment/rancher -n cattle-system
```

## Step 4: Check the Load Balancer

```bash
# For HA Rancher, traffic flows through a load balancer
# Check that all 3 Rancher replicas are in the LB target group

# For AWS ALB/NLB:
aws elbv2 describe-target-health \
  --target-group-arn <your-target-group-arn> \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Health:TargetHealth.State}'

# For nginx LB node, check the nginx config
cat /etc/nginx/nginx.conf | grep -A10 "upstream"

# Test each Rancher pod's health endpoint directly
for pod_ip in $(kubectl get pods -n cattle-system -l app=rancher \
  -o jsonpath='{.items[*].status.podIP}'); do
  echo -n "Pod ${pod_ip}: "
  curl -sk https://${pod_ip}:443/ping -o /dev/null -w "%{http_code}\n"
done
```

## Step 5: Recover from etcd Quorum Loss

If more than half the etcd members are down:

```bash
# Check which etcd members are accessible
# From a healthy node:
sudo /var/lib/rancher/rke2/bin/etcdctl member list 2>&1

# If quorum is lost, restore from the most recent snapshot
# On the surviving node:
sudo systemctl stop rke2-server

# Restore from snapshot
sudo /var/lib/rancher/rke2/bin/etcdctl snapshot restore \
  /var/lib/rancher/rke2/server/db/snapshots/etcd-snapshot-latest.db \
  --data-dir /var/lib/rancher/rke2/server/db/etcd-recovered \
  --force-new-cluster

# Move recovered data into place
sudo mv /var/lib/rancher/rke2/server/db/etcd \
        /var/lib/rancher/rke2/server/db/etcd-backup-$(date +%Y%m%d)
sudo mv /var/lib/rancher/rke2/server/db/etcd-recovered \
        /var/lib/rancher/rke2/server/db/etcd

sudo systemctl start rke2-server
```

## Step 6: Database HA Checks (External MySQL)

```bash
# For HA Rancher using MySQL Group Replication or Galera
# Check replication status (run on each MySQL node)
mysql -u root -p -e "SHOW REPLICA STATUS\G"
mysql -u root -p -e "SELECT * FROM performance_schema.replication_group_members;"

# Check Rancher can connect to all DB nodes through the VIP
nc -zv <mysql-vip> 3306
```

## Step 7: Rolling Restart for Stuck HA State

When the HA cluster is in a degraded but functional state:

```bash
# Rolling restart preserves availability during restart
kubectl rollout restart deployment/rancher -n cattle-system

# Watch pods cycling one at a time
kubectl get pods -n cattle-system -l app=rancher -w

# Verify rollout completed successfully
kubectl rollout status deployment/rancher -n cattle-system
```

## Conclusion

Rancher HA issues require analyzing both the Rancher application layer and the underlying Kubernetes control plane. Key focus areas are etcd quorum health, load balancer target group membership, leader election, and database replication. Proactive monitoring of etcd member count, database replication lag, and Rancher pod readiness will catch most HA issues before they become outages.
