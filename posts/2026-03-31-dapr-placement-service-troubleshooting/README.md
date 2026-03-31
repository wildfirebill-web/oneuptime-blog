# How to Troubleshoot Dapr Placement Service Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Placement Service, Troubleshooting, Actor, Kubernetes

Description: Diagnose and resolve common Dapr placement service issues including sidecar connection failures, Raft split-brain, actor re-distribution delays, and placement table corruption.

---

Placement service issues cause actor invocation failures, actor activation errors, and lost actor state. Diagnosing these issues requires understanding Raft consensus behavior, sidecar connection management, and actor table dissemination.

## Common Symptoms and Causes

| Symptom | Likely Cause |
|---------|-------------|
| Actor method returns 503 | Sidecar not connected to placement |
| Actor stuck in wrong instance | Placement table not disseminated |
| Placement pod restarting | OOM or CPU throttling |
| Long actor rebalancing after deploy | Raft election delay |

## Step 1 - Check Placement Pod Status

```bash
kubectl get pods -n dapr-system -l app=dapr-placement-server
kubectl describe pod dapr-placement-server-0 -n dapr-system | grep -A 10 "Events:"
```

If pods are in CrashLoopBackOff, check logs:

```bash
kubectl logs dapr-placement-server-0 -n dapr-system --previous
```

## Step 2 - Verify Raft Leader Election

In an HA setup, exactly one placement replica should be the Raft leader:

```bash
kubectl logs dapr-placement-server-0 -n dapr-system | grep -i "leader"
kubectl logs dapr-placement-server-1 -n dapr-system | grep -i "leader"
kubectl logs dapr-placement-server-2 -n dapr-system | grep -i "leader"
```

If none claim leadership, Raft cannot form a quorum. Check network connectivity between pods:

```bash
kubectl exec dapr-placement-server-0 -n dapr-system -- \
  wget -q -O- http://dapr-placement-server-1.dapr-placement-server.dapr-system.svc.cluster.local:8080/healthz
```

## Step 3 - Check Sidecar Connection

Verify that the application sidecar is connected to the placement service:

```bash
kubectl logs my-actor-pod -c daprd | grep -i "placement"
```

Look for successful connection messages:
```text
level=info msg="established connection to placement service"
```

Or connection errors:
```text
level=error msg="error connecting to placement service"
```

## Step 4 - Force Actor Rebalancing

If actors are stuck on the wrong instance after a deployment, force a rebalancing by restarting sidecars:

```bash
kubectl rollout restart deployment my-actor-service
```

The placement service will redistribute actors as sidecars reconnect.

## Step 5 - Verify Actor Registration

Check that the actor type is registered with the placement service:

```bash
curl http://localhost:3500/v1.0/metadata | jq '.actors'
```

Expected output:
```json
[{"type": "OrderActor", "count": 5}]
```

If the actor list is empty, your application is not registering actor types with Dapr.

## Step 6 - Placement Service Resource Constraints

If the placement service is OOM killed, increase its memory limit:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_placement.resources.limits.memory=512Mi \
  --set dapr_placement.resources.requests.memory=256Mi
```

## Step 7 - Raft Data Corruption

If all else fails, you can reset the Raft data directory on the placement service. This causes a brief actor re-registration period:

```bash
# WARNING: This causes all sidecars to re-register with the placement service
kubectl delete pvc -n dapr-system -l app=dapr-placement-server
kubectl rollout restart statefulset dapr-placement-server -n dapr-system
```

## Summary

Troubleshooting Dapr placement service issues follows a systematic path from pod health, through Raft leader election verification, to sidecar connection status and actor registration. Most issues resolve by ensuring the placement service has enough resources, all replicas can communicate, and sidecars are actively connected to the elected leader.
