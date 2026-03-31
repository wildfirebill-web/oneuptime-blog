# How to Configure Dapr Scheduler for High Availability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Scheduler, High Availability, Kubernetes, etcd

Description: Configure Dapr Scheduler for high availability with 3 replicas, cross-zone distribution, etcd quorum settings, and pod disruption budgets.

---

## HA Architecture for Dapr Scheduler

The Dapr Scheduler uses embedded etcd for job storage. To achieve high availability, deploy 3 or 5 replicas in a StatefulSet. etcd requires quorum - a majority of nodes must be available. With 3 replicas, 1 can fail while maintaining quorum. With 5 replicas, 2 can fail.

## Deploying in HA Mode

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_scheduler.replicaCount=3 \
  --set global.ha.enabled=true \
  --reuse-values
```

## Full HA Configuration via Helm Values

```yaml
global:
  ha:
    enabled: true

dapr_scheduler:
  replicaCount: 3
  volumeclaim:
    storageClassName: "premium-ssd"
    requestsStorage: "32Gi"
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1
      memory: 2Gi
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: dapr-scheduler
          topologyKey: kubernetes.io/hostname
```

## Pod Disruption Budget

Protect the Scheduler during voluntary disruptions like node drains:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: dapr-scheduler-pdb
  namespace: dapr-system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: dapr-scheduler
```

Apply:

```bash
kubectl apply -f dapr-scheduler-pdb.yaml
```

## Cross-Zone Distribution

Spread replicas across availability zones:

```yaml
dapr_scheduler:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: dapr-scheduler
```

## Verifying HA Health

After configuring HA, verify all nodes are members of the etcd cluster:

```bash
kubectl exec -n dapr-system dapr-scheduler-0 -- \
  etcdctl --endpoints=http://localhost:2379 member list --write-out=table
```

Simulate a failure by deleting one pod and confirming jobs still trigger:

```bash
kubectl delete pod dapr-scheduler-1 -n dapr-system
# Verify jobs still work
curl http://localhost:3500/v1.0-alpha1/jobs/test-job
```

## Summary

Configure Dapr Scheduler for high availability by deploying 3 replicas with pod anti-affinity, cross-zone topology spread, persistent storage, and a Pod Disruption Budget. Always run an odd number of replicas to maintain etcd quorum. Test HA by deleting individual pods and verifying job execution continues uninterrupted.
