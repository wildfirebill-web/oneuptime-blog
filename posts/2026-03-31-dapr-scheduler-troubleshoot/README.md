# How to Troubleshoot Dapr Scheduler Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Scheduler, Troubleshooting, Debugging, Kubernetes

Description: Troubleshoot common Dapr Scheduler issues including jobs not triggering, missed schedules, etcd failures, and connectivity problems between scheduler and sidecars.

---

## Common Dapr Scheduler Problems

The Dapr Scheduler service can encounter several categories of issues:

1. Jobs not triggering on schedule
2. Job callbacks failing to reach the application
3. etcd leader election failures
4. Connectivity issues between Scheduler and sidecar

## Diagnosing Jobs Not Triggering

First, verify the job is registered in the Scheduler:

```bash
# Check if job exists
curl http://localhost:3500/v1.0-alpha1/jobs/my-job-name
```

If the job exists but is not triggering, check Scheduler logs:

```bash
kubectl logs -n dapr-system dapr-scheduler-0 --tail=200 | grep -i "trigger\|fire\|dispatch"
```

## Checking Sidecar-to-Scheduler Connectivity

The Dapr sidecar connects to the Scheduler on port 50006. Verify connectivity:

```bash
kubectl exec -it <app-pod> -c daprd -- sh -c "nc -zv dapr-scheduler-server.dapr-system.svc.cluster.local 50006 && echo OK"
```

Check NetworkPolicies that may be blocking the connection:

```bash
kubectl get networkpolicies -A | grep -i dapr
```

## Diagnosing etcd Issues

The embedded etcd in the Scheduler can have leader election problems in HA setups:

```bash
# Check etcd leader status across all replicas
for i in 0 1 2; do
  echo "=== dapr-scheduler-$i ==="
  kubectl exec -n dapr-system dapr-scheduler-$i -- \
    etcdctl --endpoints=http://localhost:2379 endpoint status
done
```

Expected output shows one leader and others as followers.

## Fixing Missed Schedules

If the Scheduler was down during a scheduled trigger window, Dapr does not replay missed jobs by default. You can manually trigger a missed job:

```bash
# Manually invoke the job callback
curl -X POST http://localhost:3500/v1.0/invoke/my-app/method/job/my-job-name \
  -H "Content-Type: application/json" \
  -d '{"reason": "manual-replay"}'
```

## Resetting a Corrupted Scheduler

If etcd data is corrupted, reset the Scheduler (this loses all scheduled jobs):

```bash
# Delete PVCs to reset state
kubectl delete statefulset dapr-scheduler -n dapr-system
kubectl delete pvc -n dapr-system -l app=dapr-scheduler

# Redeploy
helm upgrade dapr dapr/dapr --namespace dapr-system --reuse-values
```

## Checking Application Handler Errors

If jobs are triggering but your app returns errors, check application logs:

```bash
kubectl logs <app-pod> | grep -i "job\|500\|error" | tail -50
```

Ensure your job endpoint returns HTTP 200 to acknowledge successful execution. Non-200 responses cause the Scheduler to retry.

## Summary

Troubleshooting Dapr Scheduler issues involves verifying job registration, checking connectivity on port 50006, diagnosing etcd leader election status, and examining application handler responses. Most issues stem from network policy restrictions, etcd instability in HA setups, or application handlers returning non-200 status codes.
