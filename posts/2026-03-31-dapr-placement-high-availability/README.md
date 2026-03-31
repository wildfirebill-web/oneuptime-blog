# How to Configure Dapr Placement Service for High Availability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Placement Service, High Availability, Actor, Kubernetes

Description: Configure the Dapr placement service for high availability using multiple replicas and Raft consensus to ensure actor placement continues working during node failures.

---

The Dapr placement service manages the distribution of virtual actors across application instances. If the placement service becomes unavailable, actors cannot be activated or re-distributed after a failure. High availability configuration ensures actor functionality survives control plane disruptions.

## How the Placement Service Works

The placement service maintains a consistent hash ring mapping actor types and IDs to specific application instances. When an actor is invoked, the Dapr sidecar queries the placement service to find which instance hosts that actor.

The placement service uses the Raft consensus algorithm internally to maintain a consistent view across replicas.

## Default vs. HA Deployment

By default, Dapr installs the placement service with a single replica. For production, run at least 3 replicas (the minimum for Raft quorum with fault tolerance).

## Configuring HA via Helm

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_placement.replicaCount=3 \
  --set dapr_placement.ha.enabled=true \
  --wait
```

## Manual HA Deployment

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: dapr-placement-server
  namespace: dapr-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dapr-placement-server
  template:
    metadata:
      labels:
        app: dapr-placement-server
    spec:
      containers:
        - name: dapr-placement-server
          image: daprio/dapr:1.14.0
          command: ["./placement"]
          args:
            - "--log-level"
            - "info"
            - "--enable-metrics"
            - "--replicaCount"
            - "3"
          ports:
            - containerPort: 50006
              name: raft
            - containerPort: 8080
              name: http
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

## Pod Anti-Affinity for Resilience

Distribute placement replicas across different nodes to prevent all replicas from failing together:

```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - dapr-placement-server
              topologyKey: kubernetes.io/hostname
```

## Verifying HA Status

Check that all placement replicas are running and a leader has been elected:

```bash
kubectl get pods -n dapr-system -l app=dapr-placement-server
kubectl logs dapr-placement-server-0 -n dapr-system | grep -i "leader\|raft\|elected"
```

## Behavior During Failover

When a placement replica fails:
1. The remaining replicas hold a Raft election
2. A new leader is elected (takes 1-2 seconds)
3. Sidecars reconnect to the new leader automatically
4. Actor placement resumes with a brief rebalancing period

During failover, in-flight actor invocations may return errors. Configure retries in your resiliency policy:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: actor-resiliency
spec:
  policies:
    retries:
      actorRetry:
        policy: constant
        duration: 500ms
        maxRetries: 5
  targets:
    actors:
      MyActor:
        retry: actorRetry
```

## Summary

Configuring the Dapr placement service for high availability requires at least 3 replicas with Raft consensus, pod anti-affinity to distribute replicas across nodes, and resiliency policies on the application side to handle brief disruptions during leader elections. This ensures actor functionality survives individual node failures without manual intervention.
