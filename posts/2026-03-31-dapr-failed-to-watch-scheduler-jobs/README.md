# How to Fix "Failed to Watch Scheduler Jobs" Error in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Scheduler, Job, Troubleshooting, Kubernetes

Description: Resolve the "Failed to Watch Scheduler Jobs" error in Dapr by fixing scheduler service connectivity, RBAC permissions, and version compatibility issues.

---

The "Failed to Watch Scheduler Jobs" error appears in Dapr sidecar logs when the sidecar cannot establish a watch connection with the Dapr Scheduler service, which was introduced in Dapr 1.14. This prevents scheduled jobs and reminders from firing.

## Understanding the Dapr Scheduler

The Dapr Scheduler service manages job scheduling for the Jobs API (introduced in 1.13) and Actor reminders. When the sidecar starts, it attempts to register a watch with the Scheduler to receive job triggers.

A typical error looks like:

```
error watching scheduler jobs: rpc error: code = Unavailable
desc = connection error: desc = "transport: Error while dialing:
dial tcp: lookup dapr-scheduler-server: no such host"
```

## Verifying the Scheduler Service

Check if the Scheduler deployment exists and is healthy:

```bash
kubectl get pods -n dapr-system | grep scheduler
kubectl get svc -n dapr-system | grep scheduler
```

If the scheduler is missing, you may need to upgrade Dapr:

```bash
helm upgrade dapr dapr/dapr -n dapr-system \
  --set dapr_scheduler.enabled=true \
  --reuse-values
```

## Checking RBAC Permissions

The Dapr sidecar needs permission to communicate with the scheduler. Verify the ClusterRole and bindings:

```bash
kubectl get clusterrole dapr-injector -o yaml | grep -A5 scheduler
kubectl get rolebinding -n dapr-system | grep scheduler
```

If permissions are missing, reinstall Dapr or apply the RBAC manifests:

```bash
helm upgrade dapr dapr/dapr -n dapr-system --reuse-values
```

## Network Policy Issues

If network policies are strict, ensure traffic is allowed between app pods and the scheduler service on port 50006:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dapr-scheduler
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: dapr-system
    ports:
    - port: 50006
      protocol: TCP
```

## Version Compatibility

The Scheduler service was introduced in Dapr 1.14. If you upgraded the Dapr CLI but not the Kubernetes control plane (or vice versa), there may be a version mismatch:

```bash
dapr version
kubectl describe deployment dapr-operator -n dapr-system | grep Image
```

Ensure both CLI and control plane versions match.

## Disabling the Scheduler Watch

If you are not using the Jobs API or scheduled reminders and want to suppress the error:

```yaml
annotations:
  dapr.io/disable-builtin-k8s-secret-store: "true"
```

Or configure the sidecar to not connect to the scheduler:

```bash
helm upgrade dapr dapr/dapr -n dapr-system \
  --set dapr_scheduler.replicaCount=1 \
  --reuse-values
```

## Summary

The "Failed to Watch Scheduler Jobs" error occurs when the Dapr sidecar cannot reach the Scheduler service due to missing deployments, RBAC gaps, network policies, or version mismatches. Verify the Scheduler service exists and is running, check that network policies allow traffic on port 50006, and ensure your Dapr CLI and Kubernetes control plane versions are aligned.
