# How to Troubleshoot Kubewarden Policy Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Troubleshooting, Policy as Code, Kubernetes, Debugging, Admission Control, SUSE Rancher

Description: Learn how to diagnose and fix common Kubewarden policy issues including policies not enforcing, evaluation errors, and misconfigured settings using logs, events, and kwctl.

---

Kubewarden policies can fail silently or produce unexpected results. This guide walks through the most common issues and how to diagnose them systematically.

---

## Common Issues

| Symptom | Likely Cause |
|---|---|
| Policy not enforcing | Policy not active or rules misconfigured |
| All requests rejected | Policy settings too strict or bug in logic |
| Error on every evaluation | WASM runtime issue or SDK mismatch |
| Policy works locally but fails in cluster | Missing settings or wrong registry |

---

## Step 1: Check Policy Status

```bash
# List all ClusterAdmissionPolicies
kubectl get clusteradmissionpolicy -A

# Check if a policy is active
kubectl describe clusteradmissionpolicy disallow-latest-tag

# Look for the Status field — it should be "active"
# If it shows "pending" or "error", the policy server may not have loaded it
```

A policy in `pending` state means the Policy Server is still pulling the WASM module from the registry. If it stays pending, check the Policy Server logs:

```bash
kubectl logs -n kubewarden deployment/kubewarden-policy-server-default
```

---

## Step 2: Inspect Policy Server Logs

```bash
# Follow logs for real-time evaluation output
kubectl logs -n kubewarden deployment/kubewarden-policy-server-default -f

# Filter for a specific policy
kubectl logs -n kubewarden deployment/kubewarden-policy-server-default \
  | grep "disallow-latest-tag"

# Look for error-level log entries
kubectl logs -n kubewarden deployment/kubewarden-policy-server-default \
  | grep -i "error\|panic\|failed"
```

---

## Step 3: Check Kubernetes Events

```bash
# View events in the kubewarden namespace
kubectl get events -n kubewarden --sort-by='.lastTimestamp'

# Check events for a specific policy
kubectl describe clusteradmissionpolicy disallow-latest-tag | grep -A 10 Events
```

---

## Step 4: Test with kwctl Before Deploying

The fastest way to debug a policy is to run it locally with a known request:

```bash
# Run the policy against a test request
kwctl run \
  ghcr.io/my-org/disallow-latest-tag:v0.1.0 \
  --request-path test-request.json \
  --settings-path settings.json

# Enable verbose output for detailed evaluation trace
kwctl run \
  ghcr.io/my-org/disallow-latest-tag:v0.1.0 \
  --request-path test-request.json \
  --settings-path settings.json \
  --verbose
```

---

## Step 5: Validate Policy Settings

Invalid settings cause policies to reject all requests or fail to initialize:

```bash
# Validate settings without running a full evaluation
kwctl run my-policy.wasm \
  --settings-path settings.json \
  --validate-settings

# Example output for valid settings:
# Settings validation passed

# Example output for invalid settings:
# Settings validation failed: required field 'maxReplicas' is missing
```

---

## Step 6: Check Admission Webhook Configuration

Kubewarden registers a ValidatingWebhookConfiguration. If this is misconfigured, policies either do not fire or block all traffic:

```bash
# List webhook configurations
kubectl get validatingwebhookconfigurations

# Inspect the Kubewarden webhook
kubectl describe validatingwebhookconfiguration kubewarden-policy-server-default

# Check the namespaceSelector and rules — misconfigured selectors mean
# the webhook never fires for your target resources
```

---

## Step 7: Test with a Dry-Run Request

```bash
# Perform a dry-run to see what the webhook would do without applying
kubectl apply --dry-run=server -f my-pod.yaml

# If the request is rejected, the error message will come from the policy
# If no error appears, the policy may not be matching this resource type
```

---

## Step 8: Check Policy Rules Configuration

```yaml
# Verify the rules section matches your resource
kubectl get clusteradmissionpolicy disallow-latest-tag -o yaml

# Common mistake: rules.operations missing "UPDATE"
spec:
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE"]    # <-- Only fires on CREATE, not UPDATE
```

---

## Debugging Checklist

```bash
# 1. Is the policy active?
kubectl get clusteradmissionpolicy

# 2. Is the Policy Server running?
kubectl get pods -n kubewarden

# 3. Are there errors in the Policy Server logs?
kubectl logs -n kubewarden deployment/kubewarden-policy-server-default | tail -50

# 4. Does kwctl run the policy correctly locally?
kwctl run <policy> --request-path test.json

# 5. Does the webhook configuration match your resource?
kubectl describe validatingwebhookconfiguration kubewarden-policy-server-default
```

---

## Best Practices

- Always test with `kwctl` locally before deploying to a cluster — it gives faster feedback than cluster-level testing.
- Set policies to `monitor` mode first (`spec.mode: monitor`) so you can observe what would be denied without actually blocking requests.
- Use `kubectl describe` on the policy and the webhook configuration together — most issues are in the rules or settings fields.
