# How to Debug Redis Pods in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kubernetes, Debugging, Pod, Troubleshooting

Description: A practical guide to debugging Redis pods in Kubernetes - from checking pod status and logs to diagnosing crashes, OOM kills, and connection issues.

---

Debugging Redis in Kubernetes involves multiple layers: pod scheduling, container startup, Redis process health, and network connectivity. This guide walks through a systematic approach to diagnosing common problems.

## Step 1: Check Pod Status

Start by understanding the current pod state:

```bash
kubectl get pods -n redis -l app=redis -o wide

# Get detailed pod info
kubectl describe pod redis-0 -n redis
```

Common status values to investigate:

```text
CrashLoopBackOff   - Container is crashing repeatedly
OOMKilled          - Pod ran out of memory
Pending            - Can't be scheduled (resource/taint issues)
Init:Error         - Init container failed
ImagePullBackOff   - Image can't be pulled
```

## Step 2: Read Container Logs

```bash
# Current logs
kubectl logs redis-0 -n redis

# Previous container logs (after crash)
kubectl logs redis-0 -n redis --previous

# Follow live logs
kubectl logs redis-0 -n redis -f
```

A common crash log indicating misconfiguration:

```text
# oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
Fatal error, can't open config file '/etc/redis/redis.conf': No such file or directory
```

## Step 3: Exec Into the Pod

If the pod is running, exec in to investigate the Redis process directly:

```bash
kubectl exec -it redis-0 -n redis -- bash

# Inside the pod
redis-cli ping
redis-cli info server
redis-cli info memory
redis-cli info clients
```

Expected healthy output:

```text
PONG
```

## Step 4: Diagnose OOMKilled

If Redis is being killed by OOM, check memory usage:

```bash
# Check pod resource usage
kubectl top pod redis-0 -n redis

# Check OOM events
kubectl get events -n redis --sort-by='.lastTimestamp' | grep OOM
```

Review the Redis memory configuration:

```bash
kubectl exec -it redis-0 -n redis -- redis-cli config get maxmemory
```

Output shows unset (unlimited) memory:

```text
1) "maxmemory"
2) "0"
```

Set a limit to prevent OOM kills:

```bash
kubectl exec -it redis-0 -n redis -- redis-cli config set maxmemory 512mb
kubectl exec -it redis-0 -n redis -- redis-cli config set maxmemory-policy allkeys-lru
```

## Step 5: Check Persistent Volume Issues

A common cause of Redis pod failures is PVC issues:

```bash
# Check PVC status
kubectl get pvc -n redis

# Check PV binding
kubectl describe pvc redis-data-redis-0 -n redis
```

If the PVC is in `Pending`, look for storage class or capacity issues:

```text
Events:
  Warning  ProvisioningFailed  storageclass.storage.k8s.io "fast-ssd" not found
```

## Step 6: Test Network Connectivity

From another pod, test that Redis is reachable:

```bash
kubectl run redis-debug --image=redis:7.2-alpine --restart=Never -n redis -- sleep 3600
kubectl exec -it redis-debug -n redis -- redis-cli -h redis-master.redis.svc.cluster.local -p 6379 ping
```

## Step 7: Ephemeral Debug Container (Kubernetes 1.23+)

If the Redis image lacks debugging tools, attach an ephemeral debug container:

```bash
kubectl debug -it redis-0 -n redis --image=busybox:1.36 --target=redis
```

Then run network and process diagnostics from within:

```bash
# Inside ephemeral container
nslookup redis-master.redis.svc.cluster.local
wget -qO- http://redis-master.redis.svc.cluster.local:9121/metrics 2>&1 | head -20
```

## Summary

Debugging Redis in Kubernetes starts with `kubectl describe` and `kubectl logs`, then progresses to exec-based Redis CLI diagnostics for memory, connections, and configuration issues. For persistent problems, check PVC binding, resource limits, and network policies. Use ephemeral containers when the Redis image is minimal and lacks standard debugging tools.
