# How to Troubleshoot Kubernetes Resource Quota Exceeded Errors in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Resource Quotas, Troubleshooting, Namespaces

Description: Resolve Kubernetes resource quota exceeded errors by identifying quota limits, current usage, and remediation options through Portainer.

---

When a namespace exceeds its resource quota, new pods and resources are rejected with a "exceeded quota" error. Portainer's terminal and namespace view help you identify quota limits and current usage to resolve the constraint.

## Step 1: Identify the Quota Exceeded Error

When deploying a pod that exceeds quota, the error appears in Portainer's deployment output:

\`\`\`
Error: pods "api-v2-xxx" is forbidden: exceeded quota: namespace-quota,
requested: memory=512Mi, used: memory=3.8Gi, limited: memory=4Gi
\`\`\`

## Step 2: View Current Quota Usage

\`\`\`bash
kubectl describe resourcequota -n production

# Example output:
# Name:            namespace-quota
# Resource         Used    Hard
# --------         ---     ---
# cpu              3800m   4000m
# memory           3.8Gi   4Gi
# pods             18      20
# persistentvolumeclaims  5  10
\`\`\`

## Step 3: Find Resource-Heavy Pods

\`\`\`bash
# List pods with their resource requests
kubectl get pods -n production -o custom-columns='NAME:.metadata.name,CPU:.spec.containers[*].resources.requests.cpu,MEM:.spec.containers[*].resources.requests.memory'
\`\`\`

## Step 4: Options to Resolve

**Option A: Remove unused resources**
\`\`\`bash
# Delete completed or failed pods
kubectl delete pods --field-selector=status.phase=Succeeded -n production
kubectl delete pods --field-selector=status.phase=Failed -n production
\`\`\`

**Option B: Reduce resource requests**
Reduce requests on lower-priority deployments in the namespace. Resource requests must equal actual usage only if LimitRange enforces defaults.

**Option C: Increase the quota**
\`\`\`yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: production
spec:
  hard:
    cpu: "8"
    memory: "8Gi"
    pods: "40"
\`\`\`

Apply the updated quota via Portainer's manifest interface.

## Step 5: Set LimitRange Defaults

To prevent individual pods from over-requesting resources:

\`\`\`yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
\`\`\`

## Summary

Resource quota exceeded errors are resolved by either freeing resources within the namespace or increasing the quota. Portainer's terminal gives you direct access to run kubectl describe and get commands to identify wasteful resource consumers and make targeted adjustments.
