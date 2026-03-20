# How to Deploy Cassandra on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Cassandra, Kubernetes, NoSQL, StatefulSet, Helm

Description: A practical guide to deploying an Apache Cassandra cluster on Rancher using Helm with proper storage, resource limits, and cluster validation.

## Introduction

Apache Cassandra is a distributed NoSQL database optimized for high write throughput and linear scalability. Its peer-to-peer architecture maps well to Kubernetes StatefulSets. This guide covers deploying a production-ready Cassandra cluster on Rancher.

## Prerequisites

- Rancher-managed Kubernetes cluster with at least 3 worker nodes
- `helm` and `kubectl` configured
- A StorageClass available (SSD-backed recommended for Cassandra)

## Step 1: Add Bitnami Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## Step 2: Create Namespace

```bash
kubectl create namespace cassandra
```

## Step 3: Configure Values

Cassandra requires careful resource and JVM tuning. Create a dedicated values file:

```yaml
# cassandra-values.yaml
replicaCount: 3   # One replica per availability zone

dbUser:
  user: cassandra
  password: "securepassword"

persistence:
  enabled: true
  storageClass: "longhorn"
  size: 100Gi   # Cassandra is storage-intensive

resources:
  requests:
    memory: "2Gi"
    cpu: "500m"
  limits:
    memory: "4Gi"
    cpu: "2"

jvm:
  # Cassandra JVM heap settings (25% of container memory is a good baseline)
  maxHeapSize: 1024M
  newHeapSize: 256M

cluster:
  name: "MyCluster"
  seedCount: 2   # Number of seed nodes
```

## Step 4: Deploy Cassandra

```bash
helm install cassandra bitnami/cassandra \
  --namespace cassandra \
  --values cassandra-values.yaml
```

## Step 5: Monitor Startup

Cassandra pods take time to initialize as each node joins the ring. Watch the logs closely.

```bash
# Check pod status
kubectl get pods -n cassandra -w

# Monitor ring joining process
kubectl logs -n cassandra cassandra-0 | grep "Node /"
```

## Step 6: Verify Cluster Health

Once all pods are Running, check the ring status using `nodetool`.

```bash
# Execute nodetool status inside the first pod
kubectl exec -it cassandra-0 -n cassandra -- nodetool status
```

A healthy output shows all nodes with status `UN` (Up/Normal).

## Step 7: Connect with cqlsh

```bash
# Open a CQL shell session
kubectl exec -it cassandra-0 -n cassandra -- cqlsh -u cassandra -p securepassword

-- Create a keyspace with replication factor 3
CREATE KEYSPACE myapp
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};

USE myapp;
CREATE TABLE users (id UUID PRIMARY KEY, name TEXT, email TEXT);
```

## Anti-Affinity for Resilience

Ensure Cassandra pods don't share the same node by applying pod anti-affinity rules in the values file:

```yaml
# Add under the root of cassandra-values.yaml
podAntiAffinityPreset: hard   # Forces pods onto different nodes
```

## Conclusion

You now have a three-node Cassandra cluster on Rancher. For production workloads, tune JVM heap settings to roughly 25% of the container memory limit, and use a StorageClass backed by SSDs to meet Cassandra's I/O requirements.
