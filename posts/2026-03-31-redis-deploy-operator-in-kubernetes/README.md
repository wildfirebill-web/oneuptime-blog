# How to Deploy Redis Operator in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kubernetes, Operator, Redis Operator, CRD, High Availability

Description: Deploy the Redis Operator in Kubernetes to manage Redis instances declaratively using Custom Resource Definitions for standalone, sentinel, and cluster topologies.

---

## What is a Redis Operator

A Kubernetes Operator extends the Kubernetes API with custom resources for managing Redis. It automates tasks like provisioning, configuration, upgrades, backups, and failover. Popular Redis operators include:
- **OT-Container-Kit Redis Operator** - open-source, supports standalone/cluster/sentinel
- **Redis Enterprise Operator** - for Redis Enterprise

This guide uses the OT-Container-Kit Redis Operator.

## Install the Operator with Helm

```bash
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update

helm install redis-operator ot-helm/redis-operator \
    --namespace redis-operator \
    --create-namespace \
    --version 0.15.0
```

Verify the operator is running:

```bash
kubectl get pods -n redis-operator
kubectl get crds | grep redis
```

Expected CRDs:

```text
redis.redis.redis.opstreelabs.in
rediscluster.redis.redis.opstreelabs.in
redissentinel.redis.redis.opstreelabs.in
redisreplication.redis.redis.opstreelabs.in
```

## Deploy a Standalone Redis Instance

Create `redis-standalone.yaml`:

```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: Redis
metadata:
  name: redis-standalone
  namespace: default
spec:
  kubernetesConfig:
    image: quay.io/opstree/redis:v7.0.15
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

Apply:

```bash
kubectl apply -f redis-standalone.yaml
kubectl get redis redis-standalone
kubectl get pods -l app=redis-standalone
```

## Deploy Redis Cluster

Create `redis-cluster.yaml`:

```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisCluster
metadata:
  name: redis-cluster
  namespace: default
spec:
  clusterSize: 3
  clusterVersion: v7
  kubernetesConfig:
    image: quay.io/opstree/redis:v7.0.15
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  redisExporter:
    enabled: true
    image: quay.io/opstree/redis-exporter:v1.44.0
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

Apply:

```bash
kubectl apply -f redis-cluster.yaml
kubectl get rediscluster redis-cluster
kubectl get pods -l app=redis-cluster
```

## Deploy Redis Sentinel (High Availability)

```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisSentinel
metadata:
  name: redis-sentinel
  namespace: default
spec:
  clusterSize: 3
  kubernetesConfig:
    image: quay.io/opstree/redis:v7.0.15
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
  redisSentinelConfig:
    redisReplicationName: redis-replication
    masterGroupName: mymaster
    downAfterMilliseconds: "5000"
    failoverTimeout: "10000"
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

## Check Operator-Managed Resources

```bash
# List all Redis resources
kubectl get redis,rediscluster,redissentinel,redisreplication -A

# Describe a Redis instance
kubectl describe redis redis-standalone

# View operator logs
kubectl logs -n redis-operator -l app.kubernetes.io/name=redis-operator
```

## Connect to the Managed Redis

```bash
# Exec into the Redis pod
kubectl exec -it redis-standalone-0 -- redis-cli PING

# Port-forward for local access
kubectl port-forward svc/redis-standalone 6379:6379
redis-cli -p 6379 PING
```

## Upgrading the Redis Image

Update the image tag in the CR and apply:

```bash
kubectl patch redis redis-standalone --type='json' \
    -p='[{"op":"replace","path":"/spec/kubernetesConfig/image","value":"quay.io/opstree/redis:v7.2.4"}]'
```

The operator performs a rolling restart.

## Uninstalling

```bash
kubectl delete redis redis-standalone
kubectl delete rediscluster redis-cluster
helm uninstall redis-operator -n redis-operator
```

## Summary

The Redis Operator simplifies Kubernetes-native Redis management by introducing Custom Resource Definitions for standalone, cluster, and Sentinel topologies. Operators handle pod creation, PVC management, and configuration automatically from a single declarative YAML resource. This approach reduces operational overhead compared to managing StatefulSets manually.
