# How to Test Disaster Recovery Procedures in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Disaster Recovery, Testing, Resilience

Description: Learn how to safely test Rook-Ceph disaster recovery procedures in staging and production environments to validate your DR runbooks before real incidents.

---

Testing DR procedures for Rook-Ceph before a real incident is critical. An untested runbook may contain errors, outdated commands, or incorrect assumptions. This guide covers how to simulate various failure scenarios and verify that your recovery procedures actually work.

## Setting Up a DR Test Environment

Never test DR procedures directly on production. Create a staging cluster that mirrors production topology:

```bash
# Deploy staging cluster with same Rook version
helm install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --version 1.14.0 \
  -f staging-values.yaml
```

The staging cluster should have:
- Same Rook and Ceph versions as production
- Same replication factor configuration
- Representative data volumes (does not need full production scale)
- Same network topology where possible

## Test 1: Single OSD Failure

Simulate an OSD disk failure by marking it out:

```bash
# Get OSD IDs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree

# Mark OSD 2 as out to simulate disk failure
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd out 2
```

Observe recovery behavior:

```bash
watch -n 10 "kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status"
```

Expected: Ceph begins backfilling data to other OSDs. PGs move to `degraded+remapped` then recover to `active+clean`. The RTO for this scenario should be measured and recorded.

Restore the OSD:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd in 2
```

## Test 2: Monitor Failure

Simulate a monitor failure by scaling down a mon deployment:

```bash
# Identify a mon to test with
kubectl -n rook-ceph get deploy -l app=rook-ceph-mon

# Scale down one monitor
kubectl -n rook-ceph scale deployment rook-ceph-mon-b --replicas=0
```

Observe quorum degradation:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph quorum_status
```

With two of three monitors still running, the cluster should remain operational. Measure the time for Rook to detect the failure and attempt replacement. Restore by scaling back up:

```bash
kubectl -n rook-ceph scale deployment rook-ceph-mon-b --replicas=1
```

## Test 3: Node Failure Simulation

Use kubectl to cordon and drain a storage node to simulate node failure:

```bash
NODE="node-2"
kubectl cordon $NODE
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data
```

Observe how Rook responds:

```bash
kubectl -n rook-ceph get pods -o wide | grep $NODE
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

OSD pods on the drained node will terminate. If the node hosted a monitor, Rook will attempt to move the monitor to another node. Record the recovery time. Restore:

```bash
kubectl uncordon $NODE
```

## Test 4: Namespace Deletion Recovery

This is a high-stakes test - only run in staging with a full backup taken first:

```bash
# Create backup first
velero backup create pre-dr-test --include-namespaces rook-ceph

# Simulate the disaster
kubectl delete namespace rook-ceph
```

Follow your namespace recovery runbook and measure the time to full restoration. Verify all PVCs are accessible after recovery:

```bash
kubectl get pvc -A
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

## Measuring and Recording Results

For each test, capture metrics to compare against RTO targets:

```bash
# Record start time
START=$(date +%s)

# ... perform recovery steps ...

# Record end time
END=$(date +%s)
echo "Recovery time: $((END - START)) seconds"
```

Document the results:

```text
Test: Single OSD failure
Date: 2024-01-15
Rook version: v1.14.0
Expected RTO: 15 minutes
Actual RTO: 8 minutes 32 seconds
Notes: Recovery was faster than expected. Backfill throttle setting did not impact performance.
Action items: None
```

## Automated DR Tests

Use Chaos Mesh or similar tools to automate regular DR tests:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: rook-osd-failure-test
  namespace: rook-ceph
spec:
  action: pod-failure
  mode: one
  duration: "5m"
  selector:
    namespaces:
    - rook-ceph
    labelSelectors:
      app: rook-ceph-osd
```

Schedule automated chaos tests weekly during maintenance windows and alert if recovery does not complete within RTO targets.

## Summary

Testing Rook-Ceph DR procedures involves simulating OSD failures, monitor failures, node failures, and namespace deletions in a staging environment, measuring actual recovery times against RTO targets, and documenting results to improve runbooks. Automate regular testing with chaos engineering tools and always take backups before running destructive DR tests. Only procedures that have been validated in staging should be trusted in production.
