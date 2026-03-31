# How to Configure Dapr Placement Service Raft Consensus

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Placement Service, Raft, Consensus, Configuration

Description: Configure Raft consensus parameters in the Dapr placement service to optimize leader election speed, log retention, and cluster stability in production environments.

---

The Dapr placement service uses the Raft distributed consensus algorithm to maintain a consistent actor placement table across multiple replicas. Understanding and tuning Raft parameters helps you balance leader election speed, log retention overhead, and cluster stability.

## How Raft Works in Placement Service

In a 3-replica placement service deployment:
1. One replica is elected as the Raft leader
2. All actor table updates go through the leader
3. The leader replicates updates to followers before committing
4. If the leader fails, remaining replicas elect a new leader
5. The new leader resumes handling updates after election

## Raft Quorum Requirements

Raft requires a majority of replicas to be healthy for writes to succeed:

| Replicas | Required for Quorum | Fault Tolerance |
|----------|--------------------|-----------------| 
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |

## Viewing Raft Configuration

Check the current Raft configuration via placement service logs:

```bash
kubectl logs dapr-placement-server-0 -n dapr-system | grep -i "raft\|config\|cluster"
```

## Configuring Raft Parameters via Helm

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_placement.ha.enabled=true \
  --set dapr_placement.replicaCount=3 \
  --set dapr_placement.ha.timeoutInSec=1 \
  --wait
```

## Key Raft Parameters

**heartbeatInterval**: How often the leader sends heartbeats to followers. Lower values detect failures faster but increase network traffic.

**electionTimeout**: How long a follower waits before starting an election. Should be significantly larger than heartbeatInterval to prevent spurious elections.

**snapshotInterval**: How often the Raft log is compacted into a snapshot. Reduces log size but takes CPU time.

Dapr's placement service uses defaults tuned for Kubernetes environments. Override them if needed:

```bash
kubectl exec dapr-placement-server-0 -n dapr-system -- \
  ./placement \
  --raft-heartbeat-interval=500ms \
  --raft-election-timeout=2000ms \
  --raft-snapshot-interval=120s
```

## Cluster Bootstrap

When starting a new placement cluster, all replicas must be listed so they can discover each other:

```yaml
# Placement StatefulSet with cluster peers
containers:
  - name: dapr-placement-server
    args:
      - "--initial-cluster"
      - "dapr-placement-server-0=http://dapr-placement-server-0.dapr-placement-server:8201,dapr-placement-server-1=http://dapr-placement-server-1.dapr-placement-server:8201,dapr-placement-server-2=http://dapr-placement-server-2.dapr-placement-server:8201"
```

## Checking Raft Leader

```bash
# The leader will log "leadership acquired"
for i in 0 1 2; do
  echo "=== Replica $i ==="
  kubectl logs dapr-placement-server-$i -n dapr-system | grep -i "leadership\|leader" | tail -3
done
```

## Recovering from Split-Brain

A split-brain occurs when network partitions cause two replicas to believe they are the leader. Raft prevents this by requiring quorum for writes, but network partitions can cause read inconsistencies.

To resolve a suspected split-brain:
1. Verify connectivity between all placement pods
2. Restart all placement pods simultaneously if needed
3. Check that only one pod logs "leadership acquired"

```bash
kubectl rollout restart statefulset dapr-placement-server -n dapr-system
```

## Summary

Raft consensus in the Dapr placement service provides strong consistency guarantees for actor table updates. Understanding quorum requirements, heartbeat parameters, and leader election behavior helps you configure the placement service for reliable operation and enables faster diagnosis of split-brain or election failures.
