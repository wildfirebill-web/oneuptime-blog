# How to Deploy StatefulSets via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, StatefulSet, Database, Storage

Description: Deploy and manage Kubernetes StatefulSets for databases and stateful applications via Portainer's Kubernetes interface.

## Introduction

StatefulSets are Kubernetes workload objects for managing stateful applications. Unlike Deployments, they provide stable network identities, ordered deployment, and persistent storage per pod. Portainer's Kubernetes interface supports deploying StatefulSets through YAML manifests, making it accessible without CLI access.

## When to Use StatefulSets

- Databases (PostgreSQL, MySQL, MongoDB, Cassandra)
- Distributed systems with leader election (ZooKeeper, etcd)
- Applications requiring stable hostnames
- Services with ordered startup requirements

## PostgreSQL StatefulSet

```yaml
# postgres-statefulset.yml - deploy via Portainer

apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None  # Headless service for StatefulSet DNS
  selector:
    app: postgres
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  ports:
  - port: 5432
  selector:
    app: postgres
    role: primary
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        role: primary
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          value: myapp
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - myapp
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - myapp
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 20Gi
```

## Redis Cluster StatefulSet

```yaml
# redis-statefulset.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: production
spec:
  serviceName: redis-headless
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      initContainers:
      - name: init-redis
        image: redis:7-alpine
        command:
        - sh
        - -c
        - |
          REDIS_NODES=$(nslookup redis-headless.production.svc.cluster.local | grep "Address:" | awk '{print $2}' | tr '\n' ',')
          echo "Cluster nodes: $REDIS_NODES"
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server", "--appendonly", "yes"]
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-data
          mountPath: /data
        resources:
          requests:
            memory: "128Mi"
          limits:
            memory: "256Mi"
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

## Managing StatefulSets in Portainer

Via Portainer UI: **Kubernetes > Applications**

- View StatefulSet pods with their ordinal indices (pod-0, pod-1, pod-2)
- Scale via the slider (ordered scale-up, reverse-ordered scale-down)
- View persistent volumes attached to each pod
- Access pod console for database administration

```bash
# Scale via Portainer API
curl -X PUT \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{"spec":{"replicas":3}}' \
  "https://portainer.example.com/api/endpoints/1/kubernetes/apis/apps/v1/namespaces/production/statefulsets/postgres/scale"
```

## Rolling Updates for StatefulSets

```yaml
spec:
  updateStrategy:
    type: RollingUpdate     # Default
    rollingUpdate:
      partition: 0          # Update all pods (set higher to do canary)
  
  # Or OnDelete: only update when pod is manually deleted
  updateStrategy:
    type: OnDelete
```

## Conclusion

StatefulSets deployed via Portainer provide managed, ordered, and persistent deployment for stateful applications in Kubernetes. Portainer's YAML editor and namespace management make StatefulSet deployment accessible without deep kubectl knowledge. The persistent volume claim templates ensure each pod gets its own dedicated storage, and stable network identities enable service discovery within stateful applications.
