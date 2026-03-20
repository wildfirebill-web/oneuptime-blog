# How to Debug OOMKilled Pods in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, OOMKilled, Memory, Troubleshooting

Description: Diagnose and fix OOMKilled pod errors in Kubernetes by identifying memory-hungry containers and adjusting resource limits via Portainer.

---

OOMKilled (Out Of Memory Killed) occurs when a container exceeds its memory limit and the kernel terminates it. The pod shows exit code 137. Portainer helps you identify the cause and adjust memory configuration.

## Step 1: Confirm OOMKill

In Portainer, open the pod detail and check the container's last termination reason:

\`\`\`bash
kubectl get pod <pod-name> -n production -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# Output: OOMKilled
\`\`\`

Also visible in pod events:

\`\`\`
Warning  OOMKilling  Container api exceeded memory limit
\`\`\`

## Step 2: Check Current Memory Limit

\`\`\`bash
kubectl get pod <pod-name> -n production -o jsonpath='{.spec.containers[0].resources}'
# Output: {"limits":{"memory":"256Mi"},"requests":{"memory":"128Mi"}}
\`\`\`

## Step 3: View Memory Usage Before the Kill

If you have Metrics Server or Prometheus, check peak memory usage:

\`\`\`bash
# Current memory usage (requires Metrics Server)
kubectl top pod <pod-name> -n production
kubectl top pods -n production --sort-by=memory
\`\`\`

## Step 4: Check Application Logs for Memory Patterns

View logs just before the OOMKill for clues:

\`\`\`bash
kubectl logs <pod-name> --previous -n production | tail -100
\`\`\`

Look for:
- Large data loads (queries returning millions of rows)
- Memory leak indicators (gradual heap growth)
- Burst processing (large file uploads, batch jobs)

## Step 5: Increase the Memory Limit

Edit the deployment via Portainer's manifest editor:

\`\`\`yaml
resources:
  requests:
    memory: "256Mi"   # What Kubernetes reserves
  limits:
    memory: "1Gi"     # Maximum before OOMKill (increase this)
\`\`\`

Set the limit 20-50% above observed peak usage to provide headroom.

## Step 6: Fix the Root Cause

Increasing the limit buys time but may not fix a memory leak. Profile the application:

- For Java: increase heap (-Xmx) or fix object retention issues
- For Python: use memory profiler to find leaks
- For Node.js: analyze heap snapshots with Chrome DevTools

## Summary

OOMKilled pods are resolved in two steps: immediately increase the memory limit to stop the crash loop, then profile the application to determine whether the memory usage is legitimate (set a permanent higher limit) or a leak (fix the code). Portainer's pod detail and log access give you the initial data needed for both decisions.
