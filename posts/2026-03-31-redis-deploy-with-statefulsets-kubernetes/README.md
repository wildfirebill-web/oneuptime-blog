# How to Deploy Redis with StatefulSets in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kubernetes, StatefulSet, Persistent Storage, High Availability, DevOps

Description: Deploy Redis with Kubernetes StatefulSets for stable network identities and persistent storage, with ConfigMap-based configuration and headless Service for replica discovery.

---

## Why StatefulSets for Redis

Redis requires stable hostnames and persistent storage across pod restarts. StatefulSets provide:
- **Stable pod names** - `redis-0`, `redis-1`, etc.
- **Stable DNS** - `redis-0.redis.namespace.svc.cluster.local`
- **Persistent volume claims** - each pod gets its own PVC that survives restarts

## Redis ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: default
data:
  redis.conf: |
    bind 0.0.0.0
    protected-mode no
    port 6379
    maxmemory 512mb
    maxmemory-policy allkeys-lru
    save 900 1
    save 300 10
    appendonly yes
    loglevel notice
```

## Headless Service

A headless Service provides DNS entries for each pod:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: default
  labels:
    app: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379
      name: redis
```

## ClusterIP Service for Application Access

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: default
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
  type: ClusterIP
```

## StatefulSet Manifest

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: default
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command:
            - redis-server
            - /etc/redis/redis.conf
          ports:
            - containerPort: 6379
              name: redis
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: redis-data
              mountPath: /data
            - name: redis-config
              mountPath: /etc/redis
          readinessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 15
            periodSeconds: 10
      volumes:
        - name: redis-config
          configMap:
            name: redis-config
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 5Gi
```

## Deploy to Kubernetes

```bash
kubectl apply -f redis-configmap.yaml
kubectl apply -f redis-service.yaml
kubectl apply -f redis-statefulset.yaml

# Check pod status
kubectl get pods -l app=redis

# Check PVCs
kubectl get pvc

# Verify Redis is running
kubectl exec redis-0 -- redis-cli ping
```

## Primary-Replica Setup

For a 3-pod setup (1 primary, 2 replicas), add an init container that configures replicas:

```yaml
initContainers:
  - name: init-redis
    image: redis:7-alpine
    command:
      - bash
      - -c
      - |
        set -e
        POD_INDEX=${HOSTNAME##*-}
        if [ "$POD_INDEX" != "0" ]; then
          echo "replicaof redis-0.redis.default.svc.cluster.local 6379" >> /etc/redis/redis.conf
        fi
    volumeMounts:
      - name: redis-config
        mountPath: /etc/redis
```

Then scale the StatefulSet:

```bash
kubectl scale statefulset redis --replicas=3
```

## Connect from an Application

```python
import redis

# Use the ClusterIP service
r = redis.Redis(host="redis-service.default.svc.cluster.local", port=6379)
r.ping()
```

## Checking Replication Status

```bash
# Check primary
kubectl exec redis-0 -- redis-cli INFO replication

# Check replica
kubectl exec redis-1 -- redis-cli INFO replication
```

## Summary

Redis StatefulSets in Kubernetes provide stable pod names, persistent storage via PVCs, and reliable DNS-based replica discovery through headless Services. The ConfigMap-based configuration allows updating Redis settings without rebuilding images. An init container pattern configures replica pods to follow the primary (`redis-0`) automatically on startup.
