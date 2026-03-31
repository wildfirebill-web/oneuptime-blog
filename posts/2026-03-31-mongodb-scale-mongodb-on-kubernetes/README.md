# How to Scale MongoDB on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Kubernetes, StatefulSet, Scaling, Replica Set

Description: Learn how to scale MongoDB replica sets on Kubernetes by adding members, configuring read preferences, and handling shard cluster expansion.

---

## Introduction

Scaling MongoDB on Kubernetes involves both horizontal scaling (adding replica set members or shards) and vertical scaling (increasing CPU and memory). Because MongoDB uses StatefulSets, scaling requires careful orchestration to maintain data consistency and cluster health.

## Scaling a Replica Set

MongoDB replica sets can be expanded by increasing the StatefulSet replica count and adding new members to the replica set configuration.

### Update the StatefulSet

```bash
# Scale StatefulSet from 1 to 3 replicas
kubectl scale statefulset mongodb --replicas=3 -n mongodb

# Verify new pods are running
kubectl get pods -n mongodb -w
```

### Add Members to the Replica Set

Once the new pods are running and synced:

```javascript
// Connect to primary
rs.add("mongodb-1.mongodb:27017")
rs.add("mongodb-2.mongodb:27017")

// Verify all members are healthy
rs.status()
```

## Configuring Read Preferences for Read Scaling

```javascript
// Node.js example - distribute reads across secondaries
const client = new MongoClient(uri, {
  readPreference: "secondaryPreferred",
  readPreferenceTags: [{ region: "us-east" }]
})
```

## Adjusting Resource Requests on Scale

Update the StatefulSet resource profile before scaling:

```yaml
spec:
  template:
    spec:
      containers:
        - name: mongodb
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
```

```bash
# Apply the resource update
kubectl apply -f mongodb-statefulset.yaml

# Rolling restart to apply changes
kubectl rollout restart statefulset/mongodb -n mongodb
```

## Scaling with the MongoDB Community Operator

If you use the MongoDB Community Operator, scaling is declarative:

```yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongodb-replica-set
  namespace: mongodb
spec:
  members: 5
  type: ReplicaSet
  version: "7.0.0"
```

```bash
kubectl apply -f mongodb-community.yaml
# The operator handles pod creation and rs.add() automatically
```

## Scaling a Sharded Cluster

For a sharded MongoDB cluster, add shards by deploying additional shard StatefulSets:

```bash
# Deploy a new shard StatefulSet (shard2)
kubectl apply -f shard2-statefulset.yaml

# Add the new shard to the cluster via mongos
kubectl exec -it mongos-0 -n mongodb -- mongosh

# In mongosh
sh.addShard("shard2/mongodb-shard2-0.mongodb-shard2:27017")
sh.status()
```

## Monitoring Scaling Operations

```bash
# Watch pod readiness during scale-up
kubectl get pods -n mongodb -l app=mongodb -w

# Check replica set member states
kubectl exec -it mongodb-0 -n mongodb -- mongosh --eval "rs.status()" | grep -E "name|stateStr"

# Watch for initial sync completion
kubectl logs mongodb-2 -n mongodb | grep "initial sync"
```

## Summary

Scaling MongoDB on Kubernetes requires coordinating StatefulSet replica counts with MongoDB replica set configuration. For managed deployments, the MongoDB Community Operator simplifies scaling to a single field change. When scaling read capacity, configure `secondaryPreferred` read preferences to distribute query load. For write throughput, horizontal sharding is the path forward, but it requires planning the shard key carefully before data grows too large.
