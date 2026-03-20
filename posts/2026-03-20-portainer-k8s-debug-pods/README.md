# How to Debug Kubernetes Pod Failures in Portainer - K8s Pods

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Debugging, Troubleshooting, Operations

Description: Use Portainer's Kubernetes interface to debug failing pods by viewing events, logs, and resource states.

## Introduction

Pod failures in Kubernetes can stem from image pull errors, resource exhaustion, configuration mistakes, or application crashes. Portainer provides a web-based interface to diagnose these issues without needing direct kubectl access.

## Step 1: Identify Failing Pods via Portainer

Navigate to: **Kubernetes > Applications**

Look for pods in states:
- **Error**: Container exited with non-zero code
- **CrashLoopBackOff**: Container repeatedly crashing
- **OOMKilled**: Out of memory
- **ImagePullBackOff**: Can't pull container image
- **Pending**: Can't be scheduled

## Step 2: View Pod Events

In Portainer: **Kubernetes > Applications > Pod > Events**

Or via kubectl:
```bash
# View events for a specific pod

kubectl describe pod failing-pod-xxx -n production

# View events for all pods in a namespace
kubectl get events -n production --sort-by='.lastTimestamp'

# Watch events in real-time
kubectl get events -n production -w
```

## Step 3: Check Pod Logs

In Portainer: **Kubernetes > Applications > Pod > Logs**

```bash
# Current logs
kubectl logs failing-pod-xxx -n production

# Previous container instance (after crash)
kubectl logs failing-pod-xxx -n production --previous

# Logs from a specific container in a multi-container pod
kubectl logs pod-xyz -c sidecar-container -n production

# Tail logs with timestamps
kubectl logs failing-pod-xxx -n production --timestamps --tail=100
```

## Step 4: Interactive Debugging

Portainer provides a web console: **Kubernetes > Applications > Pod > Console**

```bash
# Or via kubectl
kubectl exec -it failing-pod-xxx -n production -- /bin/sh

# For pods that have crashed and can't exec, use a debug container
kubectl debug -it failing-pod-xxx \
  --image=busybox \
  --copy-to=debug-pod \
  -n production

# Debug directly on a node
kubectl debug node/worker1 -it --image=ubuntu
```

## Step 5: Check Resource Usage

```bash
# Check pod resource consumption
kubectl top pod failing-pod-xxx -n production

# Check node resources
kubectl top nodes

# Get detailed pod resource requests/limits
kubectl get pod failing-pod-xxx -n production -o json \
  | python3 -c "
import sys, json
pod = json.load(sys.stdin)
for c in pod['spec']['containers']:
    name = c['name']
    req = c.get('resources', {}).get('requests', {})
    lim = c.get('resources', {}).get('limits', {})
    print(f'{name}: requests={req}, limits={lim}')
"
```

## Common Failure Patterns and Solutions

```bash
# Pattern: Pod in Pending state
kubectl describe pod pending-pod | grep -A5 "Events"
# Look for: "Insufficient cpu", "Insufficient memory", "No nodes available"

# Solution: Check node resources or adjust requests
kubectl describe nodes | grep -A5 "Allocated resources"

# Pattern: OOMKilled
kubectl describe pod oom-pod | grep -A2 "Last State"
# Shows: Reason: OOMKilled

# Solution: Increase memory limit
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"512Mi"}}}]}}}}'

# Pattern: ImagePullBackOff
kubectl describe pod image-err-pod | grep "Failed to pull image"
# Check: image name, tag, and registry credentials
```

## Using Portainer's YAML Editor for Fixes

1. Go to **Kubernetes > Applications > Your Deployment**
2. Click **Edit** or the YAML button
3. Make changes (update image, increase resources, fix env vars)
4. Click **Update the application**

## Creating Temporary Debug Deployments

```yaml
# debug-deployment.yml - deploy via Portainer for troubleshooting
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debug-tools
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: debug-tools
  template:
    metadata:
      labels:
        app: debug-tools
    spec:
      containers:
      - name: debug
        image: nicolaka/netshoot:latest  # Network debugging tools
        command: ["sleep", "infinity"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
```

## Conclusion

Portainer's Kubernetes interface provides accessible debugging tools for pod failures through log viewing, event inspection, and interactive console access. For most common failures (image issues, resource exhaustion, configuration errors), the information needed to diagnose and fix issues is available directly in the Portainer UI without requiring kubectl knowledge or CLI access.
