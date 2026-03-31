# How to Debug Kubernetes Pod Failures in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Debugging, Pod, Troubleshooting, DevOps

Description: Use Portainer's Kubernetes interface to diagnose and resolve pod failures - from Pending states and scheduling issues to container crashes and startup errors.

---

Pod failures in Kubernetes fall into a predictable set of categories. Portainer's namespace and pod views surface the information you need to diagnose each type quickly, without having to switch between multiple kubectl commands.

## Pod Failure Categories

| State | Common Causes |
|---|---|
| Pending | Insufficient resources, PVC not bound, node selector mismatch |
| CrashLoopBackOff | Application error, missing env var, wrong command |
| ImagePullBackOff | Wrong image name, missing registry credentials |
| OOMKilled | Container exceeded memory limit |
| Error | Container exited with non-zero code |
| Evicted | Node ran out of resources |

## Step 1: Identify the Failing Pod

In Portainer, go to **Kubernetes > Namespaces > [namespace] > Pods**. Filter by status - unhealthy pods show red or yellow status indicators.

## Step 2: View Pod Events

The most informative first step is checking pod events. In Portainer, open the pod detail view and scroll to the **Events** section. Events show:

- Scheduling decisions (which node was selected)
- Image pull attempts and errors
- Container start/crash events
- Volume mount failures

```bash
# Equivalent kubectl command

kubectl describe pod <pod-name> -n <namespace>
```

## Step 3: Check Container Logs

For pods in CrashLoopBackOff, view the previous container's logs to catch the crash reason:

```bash
# View logs from the previous (crashed) container instance
kubectl logs <pod-name> -n <namespace> --previous
```

In Portainer, use the **Logs** tab on the pod detail page. Toggle "Previous" to see logs from the last failed instance.

## Step 4: Check Resource Availability

For Pending pods, check whether nodes have sufficient resources:

```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl get events -n <namespace> --field-selector reason=FailedScheduling
```

If pods are pending due to resource pressure, scale down lower-priority workloads or add nodes.

## Step 5: Check Node Conditions

Identify problematic nodes via Portainer's terminal:

```bash
kubectl get nodes
kubectl describe node <node-name> | grep -A 10 "Conditions:"
```

Nodes in `NotReady`, `MemoryPressure`, or `DiskPressure` conditions cannot schedule new pods.

## Step 6: Interactive Debugging

For pods that start successfully but misbehave, use Portainer's **Console** feature to exec into the container:

1. Open the pod in Portainer
2. Click **Console**
3. Run diagnostic commands inside the running container

```bash
# Inside the container
env | grep DATABASE   # Verify env vars
curl http://postgres-service:5432  # Test connectivity
ls /app/config        # Verify file mounts
```

## Summary

Portainer's pod detail view, event log, and console access cover the majority of Kubernetes debugging workflows. Events reveal scheduling and startup issues, previous-instance logs reveal application crashes, and the interactive console lets you probe a running container's environment without writing scripts.
