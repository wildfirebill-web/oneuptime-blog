# How to Optimize Dapr Actor Placement Efficiency

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Placement, Performance, Optimization

Description: Optimize Dapr actor placement efficiency by tuning placement service replication, actor scan intervals, idle timeout settings, and actor redistribution during scaling events.

---

## Overview

The Dapr placement service distributes actor instances across pods using a consistent hashing ring. Inefficient placement leads to hot spots, slow actor activation, and uneven load distribution. This guide covers tuning placement to maximize actor throughput and balance.

## Scaling the Placement Service

Run multiple placement service replicas for fault tolerance and higher throughput:

```bash
# Scale placement service (uses Raft for consensus)
kubectl scale statefulset dapr-placement-server -n dapr-system --replicas=3

# Verify all replicas are running and one is the leader
kubectl get pods -n dapr-system -l app=dapr-placement-server
kubectl logs -n dapr-system dapr-placement-server-0 | grep -i "leader\|raft"
```

## Tuning Actor Scan and Idle Timeout

Configure actor garbage collection to remove inactive actors from the placement ring:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: actor-config
spec:
  actor:
    actorIdleTimeout: "1h"
    actorScanInterval: "30s"
    drainOngoingCallTimeout: "60s"
    drainRebalancedActors: true
    reentrancyConfig:
      enabled: false
    remindersStoragePartitions: 0
```

Apply to your service:

```yaml
annotations:
  dapr.io/config: "actor-config"
```

## Actor Type Registration

Register only the actor types your service hosts to keep the placement ring lean:

```javascript
const { DaprServer } = require('@dapr/dapr');
const server = new DaprServer();

// Only register actor types this pod hosts
server.actor.registerActor(OrderActor);
// Do NOT register actors from other services on this pod

await server.actor.init();
```

Verify registration:

```bash
curl http://localhost:3500/v1.0/actors | jq '.activeActorsCount'
```

## Reducing Placement Rebalancing During Scaling

Enable graceful actor drain during rolling updates to prevent placement churn:

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 90
      containers:
      - name: order-service
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
```

The `drainOngoingCallTimeout` and `drainRebalancedActors` settings in the actor config allow actors to finish in-progress calls before being migrated.

## Monitoring Actor Placement

Check placement distribution across pods:

```bash
# View actor counts per pod
curl http://localhost:3500/v1.0/metadata | jq '.activeActorsCount'

# Check placement service health
kubectl exec -n dapr-system dapr-placement-server-0 -- \
  wget -qO- http://localhost:8080/healthz

# Watch placement logs during a scale event
kubectl logs -n dapr-system dapr-placement-server-0 -f | grep "disseminate"
```

## Partitioning Actor Reminders

For services with many actors using reminders, increase reminder storage partitions to reduce contention on the state store:

```yaml
spec:
  actor:
    remindersStoragePartitions: 16
```

This distributes reminder storage across 16 keys instead of one, reducing hot key contention in Redis:

```bash
# Verify partitions in Redis
redis-cli KEYS "reminder-*" | wc -l
```

## Summary

Dapr actor placement efficiency depends on running a replicated placement service for fault tolerance, configuring appropriate idle timeouts to garbage collect inactive actors, enabling graceful drain during rolling updates to minimize rebalancing churn, and partitioning actor reminder storage for high-actor-count workloads. Monitor active actor counts via the metadata API and placement service logs to detect hot spots and uneven distribution.
