# How to Edit a Running Kubernetes Application in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Applications, Deployment, DevOps

Description: Learn how to edit and update a running Kubernetes application in Portainer without downtime using rolling updates.

## Introduction

Updating a running Kubernetes application in Portainer is straightforward — you can edit the application via form or YAML, and Portainer applies the changes using a rolling update strategy. This guide covers editing running applications, understanding the update process, and best practices for zero-downtime updates.

## Prerequisites

- Portainer with Kubernetes environment
- A running application to update

## Step 1: Navigate to the Application

1. Select your Kubernetes environment in Portainer
2. Click **Applications** in the sidebar
3. Find the application you want to update
4. Click on the application name

## Step 2: Edit via Form

1. Click **Edit** (pencil icon)
2. Make changes to the form fields:

**Common edits:**
- Change the image tag (version update)
- Add or modify environment variables
- Adjust resource limits
- Change replica count
- Update volume mounts

3. Click **Update the application**

## Step 3: Edit via YAML

For more complex changes:

1. Click the **YAML** tab on the application detail page
2. Directly edit the YAML:

```yaml
# Change image version
containers:
  - name: app
    image: registry.company.com/myapp:v2.1.0    # Changed from v2.0.0

# Add new environment variable
    env:
      - name: NEW_FEATURE
        value: "enabled"

# Update resource limits
    resources:
      limits:
        memory: 1Gi     # Increased from 512Mi
```

3. Click **Update**

## Step 4: Understanding Rolling Updates

By default, Kubernetes uses RollingUpdate strategy:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during update
      maxUnavailable: 0    # Never reduce below desired replicas (zero-downtime)
```

During an update with 3 replicas:

```
Initial:   pod-1 (old), pod-2 (old), pod-3 (old)
Step 1:    pod-1 (old), pod-2 (old), pod-3 (old), pod-4 (new)  ← Extra pod created
           pod-1 (old), pod-2 (old), pod-3 (terminating), pod-4 (new)
Step 2:    pod-1 (old), pod-2 (old), pod-5 (new), pod-4 (new)
           pod-1 (old), pod-2 (terminating), pod-5 (new), pod-4 (new)
Step 3:    pod-1 (old), pod-6 (new), pod-5 (new), pod-4 (new)
           pod-6 (new), pod-5 (new), pod-4 (new)  ← Update complete
```

## Step 5: Monitor the Update

```bash
# Watch the rollout
kubectl rollout status deployment/my-app -n production

# Output while updating:
# Waiting for deployment "my-app" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "my-app" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "my-app" successfully rolled out

# Watch pods
kubectl get pods -n production -l app=my-app -w
```

In Portainer, the application detail shows pods with their status updating in real time.

## Step 6: Rollback if Something Goes Wrong

```bash
# Rollback to previous version
kubectl rollout undo deployment/my-app -n production

# Check rollout history
kubectl rollout history deployment/my-app -n production

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=3 -n production
```

In Portainer: find the previous image tag or configuration and update again.

## Step 7: Common Application Updates

### Image Update (Deployment)

```bash
# Update image via kubectl (equivalent to Portainer form edit)
kubectl set image deployment/my-app \
  app=registry.company.com/myapp:v2.1.0 \
  -n production
```

### Scale Up

```bash
kubectl scale deployment my-app --replicas=5 -n production
```

### Environment Variable Update

```bash
kubectl set env deployment/my-app \
  NODE_ENV=production \
  LOG_LEVEL=warn \
  -n production
```

### Force Restart (Redeploy with Same Image)

```bash
kubectl rollout restart deployment/my-app -n production
```

## Step 8: Patch Strategy

For partial YAML updates without replacing the entire resource:

```bash
# Add or update a single field
kubectl patch deployment my-app -n production \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/replicas", "value": 5}]'

# Add an annotation
kubectl patch deployment my-app -n production \
  --type='merge' \
  -p='{"metadata":{"annotations":{"deployment.kubernetes.io/description":"v2.1.0 release"}}}'
```

## Step 9: Best Practices for Zero-Downtime Updates

1. **Always set `maxUnavailable: 0`** — Prevents availability loss during updates
2. **Configure readiness probes** — New pods don't receive traffic until healthy
3. **Set `minReadySeconds`** — Add a buffer before Kubernetes considers a pod ready

```yaml
spec:
  minReadySeconds: 30    # Wait 30s before marking new pod as available
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - name: app
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

## Conclusion

Editing running Kubernetes applications in Portainer is safe and non-disruptive when the deployment strategy is configured correctly. Use the form UI for simple updates like image tags and replica counts, and the YAML editor for structural changes. Always monitor the rolling update progress and keep rollback procedures ready for when updates don't go as planned.
