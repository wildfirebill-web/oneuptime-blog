# How to Troubleshoot Rancher Webhook Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Webhooks

Description: Diagnose and fix Rancher webhook failures that block resource creation, updates, and deletions in Rancher-managed Kubernetes clusters.

## Introduction

Rancher deploys admission webhooks to enforce policies and manage cluster resources. When these webhooks fail — due to pod crashes, TLS errors, or timeout issues — Kubernetes API calls are rejected, preventing users from creating or modifying resources. This guide explains how to identify and resolve webhook failures quickly.

## Understanding Rancher Webhooks

Rancher installs two key admission webhooks:

1. **rancher-webhook** — Validates and mutates Rancher resources (clusters, projects, users).
2. **cattle-webhook** — Enforces PSPs, namespace restrictions, and RBAC policies.

```bash
# List all webhooks in the cluster
kubectl get validatingwebhookconfiguration
kubectl get mutatingwebhookconfiguration
```

## Step 1: Identify Webhook Failure Messages

When a webhook fails, Kubernetes returns an error like:

```
Error from server (InternalError): error when creating "manifest.yaml":
Internal error occurred: failed calling webhook
"rancher.cattle.io": Post "https://rancher-webhook.cattle-system.svc:443/v1/webhook/validation/...":
dial tcp: connect: connection refused
```

```bash
# Check the webhook deployment status
kubectl get pods -n cattle-system -l app=rancher-webhook
kubectl get pods -n cattle-system -l app=cattle-webhook

# If pods are not Running, get details
kubectl describe pod -n cattle-system -l app=rancher-webhook
kubectl logs -n cattle-system -l app=rancher-webhook --tail=100
```

## Step 2: Check Webhook Service Endpoints

```bash
# Verify the webhook service exists and has endpoints
kubectl get service -n cattle-system rancher-webhook
kubectl get endpoints -n cattle-system rancher-webhook

# If endpoints are empty, the webhook pod selector doesn't match
kubectl describe service -n cattle-system rancher-webhook | grep Selector
kubectl get pods -n cattle-system --show-labels | grep rancher-webhook
```

## Step 3: Test Webhook TLS Connectivity

```bash
# Get the webhook's CA bundle
kubectl get validatingwebhookconfiguration rancher.cattle.io \
  -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | base64 -d \
  | openssl x509 -noout -dates

# Test connectivity to the webhook service
kubectl run webhook-test --rm -it \
  --image=nicolaka/netshoot \
  --restart=Never \
  -- curl -vk https://rancher-webhook.cattle-system.svc:443/healthz
```

## Step 4: Check Webhook Timeout Configuration

```bash
# View the timeoutSeconds setting on webhooks
kubectl get validatingwebhookconfiguration rancher.cattle.io -o json \
  | jq '.webhooks[] | {name: .name, timeoutSeconds: .timeoutSeconds, failurePolicy: .failurePolicy}'

# Default timeout is 10 seconds — if the webhook pod is slow, increase it
kubectl patch validatingwebhookconfiguration rancher.cattle.io \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/timeoutSeconds","value":30}]'
```

## Step 5: Temporarily Bypass Webhooks (Emergency Only)

**Warning**: Only do this in an emergency and restore the webhook immediately after.

```bash
# Check the failurePolicy — if "Ignore", failures won't block API calls
# If "Fail", API calls are blocked when the webhook is unreachable

# Option 1: Change failurePolicy to Ignore temporarily
kubectl patch validatingwebhookconfiguration rancher.cattle.io \
  --type='json' \
  -p='[{"op":"replace","path":"/webhooks/0/failurePolicy","value":"Ignore"}]'

# Option 2: Delete the webhook configuration entirely (VERY RISKY)
# Only do this if you need to unblock operations and will re-install immediately
kubectl delete validatingwebhookconfiguration rancher.cattle.io
```

## Step 6: Restart and Redeploy the Webhook

```bash
# Restart the webhook deployment
kubectl rollout restart deployment/rancher-webhook -n cattle-system

# Watch rollout status
kubectl rollout status deployment/rancher-webhook -n cattle-system

# If the webhook was accidentally deleted, re-install it via Rancher
# The rancher-webhook is managed by Rancher itself and will be re-created
# Force reconciliation by restarting Rancher
kubectl rollout restart deployment/rancher -n cattle-system
```

## Step 7: Check for Resource Exhaustion

```bash
# Webhook pods may OOMKill under heavy load
kubectl top pod -n cattle-system -l app=rancher-webhook

# Increase memory limits
kubectl patch deployment rancher-webhook -n cattle-system --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"512Mi"}
]'
```

## Step 8: Verify After Fix

```bash
# Test that webhook is now accepting requests
kubectl create namespace test-webhook-ns
kubectl delete namespace test-webhook-ns

# Check webhook logs for successful validations
kubectl logs -n cattle-system -l app=rancher-webhook --tail=50 | grep -i "allowed\|denied"
```

## Conclusion

Rancher webhook failures block Kubernetes API operations and can halt your DevOps workflows. The most common causes are crashed webhook pods, TLS certificate mismatches, and timeout issues. By combining pod status checks, service endpoint verification, and timeout adjustments, you can quickly restore webhook functionality. Always monitor webhook pod health as part of your cluster observability strategy.
