# How to Troubleshoot Kubewarden Policy Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, Policy, Troubleshooting, Debugging

Description: A comprehensive guide to diagnosing and resolving common Kubewarden policy failures, including installation issues, policy activation problems, and unexpected denials.

## Introduction

Kubewarden policy failures can manifest in several ways: policies that fail to activate, policies that unexpectedly deny legitimate workloads, or policies that fail to block clearly non-compliant resources. Quickly identifying the root cause requires understanding Kubewarden's component architecture and knowing where to look for diagnostic information.

This guide provides a systematic approach to troubleshooting Kubewarden issues.

## Prerequisites

- Kubewarden installed on the cluster
- `kubectl` with cluster-admin access
- `kwctl` CLI for local policy testing

## Quick Diagnostics Overview

```bash
# 1. Check all Kubewarden pods are running
kubectl get pods -n kubewarden

# 2. Check PolicyServer status
kubectl get policyserver

# 3. Check all policy statuses
kubectl get clusteradmissionpolicies
kubectl get admissionpolicies -A

# 4. Check recent policy events
kubectl get events -A --field-selector reason=PolicyViolation

# 5. Check Kubewarden logs
kubectl logs -n kubewarden -l app=kubewarden-controller --tail=50
```

## Troubleshooting Policy Activation Issues

### Symptom: Policy Stuck in "PolicyActive: False"

```bash
# Check the policy status conditions
kubectl describe clusteradmissionpolicy my-policy | grep -A 20 "Conditions:"

# Look for error messages in controller logs
kubectl logs -n kubewarden \
  -l app=kubewarden-controller \
  | grep -i "error\|failed\|policy-name"
```

### Common Causes of Activation Failure

**1. Policy Wasm module not accessible**

```bash
# Check if the Wasm module URL is reachable
# Test from within the policy server pod
kubectl exec -n kubewarden \
  $(kubectl get pods -n kubewarden \
    -l app=kubewarden-policy-server-default \
    -o jsonpath='{.items[0].metadata.name}') \
  -- wget -q --spider \
  ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

# Check for image pull errors
kubectl describe policyserver default -n kubewarden \
  | grep -A 10 "Events:"
```

**2. Invalid policy settings**

```bash
# Validate the settings before deploying
kwctl validate-settings \
  registry://ghcr.io/kubewarden/policies/allowed-image-repositories:v0.1.0 \
  --settings-json '{"allowedRegistries": []}'

# Expected output for invalid settings: error message
```

**3. PolicyServer not running**

```bash
# Check the PolicyServer referenced by the policy
kubectl get policyserver default

# Check PolicyServer pods
kubectl get pods -n kubewarden \
  -l app=kubewarden-policy-server-default

# Restart the PolicyServer if needed
kubectl rollout restart deployment \
  kubewarden-policy-server-default \
  -n kubewarden
```

## Troubleshooting Unexpected Denials

### Getting the Denial Reason

```bash
# When a resource is denied, the error message tells you why
kubectl apply -f my-pod.yaml 2>&1

# Expected output with details:
# Error from server: error when creating "my-pod.yaml":
# admission webhook "clusterwide-no-privileged.kubewarden.admission" denied the request:
# Container 'app' is running as privileged
```

### Tracing a Denial to a Specific Policy

```bash
# Find which policy denied the request
# Look at the webhook name in the error message
# Format: <policy-name>.<namespace>.kubewarden.admission

# Get events for the denied resource
kubectl get events -n my-namespace \
  --field-selector involvedObject.name=my-pod \
  --sort-by='.lastTimestamp'

# Check policy server logs for the denial
kubectl logs -n kubewarden \
  -l app=kubewarden-policy-server-default \
  | grep -i "my-pod\|deny\|reject"
```

### Testing a Specific Resource Against Policies

```bash
# Get the resource as JSON and test it locally
kubectl get pod existing-pod -n my-namespace -o json > /tmp/pod.json

# Wrap it in an admission request format
cat <<EOF > /tmp/test-request.json
{
  "request": {
    "uid": "debug-test",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "namespace": "my-namespace",
    "object": $(cat /tmp/pod.json)
  }
}
EOF

# Test against the denying policy
kwctl run \
  registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0 \
  --request-path /tmp/test-request.json
```

## Troubleshooting Policy Not Blocking

### Symptom: Policy Active But Not Blocking Non-Compliant Resources

```bash
# Check the policy's mode - monitor mode won't block
kubectl get clusteradmissionpolicy my-policy \
  -o jsonpath='{.spec.mode}'
# Should output "protect", not "monitor"

# Check the policy's rules match the resource being submitted
kubectl get clusteradmissionpolicy my-policy \
  -o jsonpath='{.spec.rules}'

# Verify the namespace isn't excluded
kubectl get clusteradmissionpolicy my-policy \
  -o jsonpath='{.spec.namespaceSelector}'
```

### Verify Webhook Configuration

```bash
# Check that the ValidatingWebhookConfiguration exists
kubectl get validatingwebhookconfiguration | grep kubewarden

# Verify the webhook is targeting the correct resources
kubectl describe validatingwebhookconfiguration \
  clusterwide-my-policy.kubewarden.admission

# Check the webhook's namespace selector
kubectl get validatingwebhookconfiguration \
  clusterwide-my-policy.kubewarden.admission \
  -o jsonpath='{.webhooks[0].namespaceSelector}'
```

## Troubleshooting PolicyServer Performance

### Symptom: Slow Admission Webhook Responses

```bash
# Check PolicyServer resource usage
kubectl top pods -n kubewarden

# Check for OOM events
kubectl get events -n kubewarden \
  --field-selector reason=OOMKilled

# Increase PolicyServer resources if needed
kubectl patch policyserver default -n kubewarden \
  --type=merge \
  -p '{"spec":{"resources":{"limits":{"memory":"2Gi","cpu":"2"}}}}'
```

### Webhook Timeout Issues

```bash
# Check the webhook timeout configuration
kubectl get validatingwebhookconfiguration \
  -o jsonpath='{range .items[?(@.metadata.labels.kubewarden)]}{.metadata.name}: timeout={.webhooks[0].timeoutSeconds}{"\n"}{end}'

# Kubewarden uses 10 seconds by default
# Increase if policies are too slow
```

## Recovering from a Broken PolicyServer

If the PolicyServer crashes and blocks all admissions:

```bash
# EMERGENCY: Delete the ValidatingWebhookConfiguration
# This disables ALL Kubewarden policies temporarily
# Only do this in a critical production incident

kubectl delete validatingwebhookconfiguration \
  $(kubectl get validatingwebhookconfiguration | grep kubewarden | awk '{print $1}')

# Fix the issue, then recreate the PolicyServer
kubectl rollout restart deployment \
  kubewarden-policy-server-default \
  -n kubewarden

# Restart the controller to recreate webhooks
kubectl rollout restart deployment \
  kubewarden-controller \
  -n kubewarden
```

## Enabling Debug Logging

```bash
# Enable debug logging on the PolicyServer
kubectl patch policyserver default -n kubewarden \
  --type=merge \
  -p '{"spec":{"env":[{"name":"KUBEWARDEN_LOG_LEVEL","value":"debug"}]}}'

# View debug logs
kubectl logs -n kubewarden \
  -l app=kubewarden-policy-server-default \
  --follow \
  | grep -i "debug\|policy\|deny\|allow"
```

## Conclusion

Troubleshooting Kubewarden policy failures requires a layered approach: check component health first, then policy activation status, then examine specific denials or missed blocks. The combination of `kubectl describe`, event watching, log analysis, and `kwctl` local testing covers the vast majority of issues you will encounter. By maintaining good observability practices and understanding the relationship between PolicyServers, webhook configurations, and policy resources, you can quickly diagnose and resolve any Kubewarden issue.
