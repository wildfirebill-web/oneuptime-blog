# How to Configure Fleet Auto-Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Auto-Update

Description: Learn how to configure Fleet's automatic update mechanisms to keep your Kubernetes deployments synchronized with Git changes without manual intervention.

## Introduction

Fleet's auto-update functionality is at the heart of its GitOps model. When changes are pushed to a tracked Git repository, Fleet automatically detects those changes and applies them to the target clusters. Understanding how to configure polling intervals, webhook triggers, and update strategies helps you balance between real-time responsiveness and cluster stability.

## Prerequisites

- Fleet installed and operational
- GitRepo resources configured
- `kubectl` access to Fleet manager
- Optionally: a publicly accessible Fleet endpoint for webhooks

## How Fleet Auto-Updates Work

Fleet uses two mechanisms to detect changes:

1. **Polling**: Fleet periodically polls the Git repository for new commits (default: every 15 seconds)
2. **Webhooks**: Git providers can push notifications to Fleet when commits are made (near-instant)

## Configuring Polling Intervals

### Setting a Custom Polling Interval

```yaml
# gitrepo-polling.yaml

apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/my-app
  branch: main

  # Set polling interval (default: 15s)
  # Use standard Go duration format: 15s, 1m, 5m, 1h
  pollingInterval: 30s

  targets:
    - clusterSelector: {}
```

### Disabling Polling (Webhook-Only Mode)

```yaml
# gitrepo-no-polling.yaml
spec:
  repo: https://github.com/my-org/my-app
  branch: main

  # Disable automatic polling - rely on webhooks only
  disablePolling: true
```

## Setting Up Webhook-Based Updates

### Step 1: Expose the Fleet Webhook Service

Fleet includes a webhook server that receives push notifications from Git providers. Expose it externally:

```yaml
# fleet-webhook-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fleet-webhook
  namespace: cattle-fleet-system
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: fleet-webhook.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gitjob
                port:
                  number: 80
  tls:
    - hosts:
        - fleet-webhook.example.com
      secretName: fleet-webhook-tls
```

### Step 2: Configure GitHub Webhook

1. Go to your GitHub repository
2. Navigate to **Settings > Webhooks > Add webhook**
3. Set **Payload URL**: `https://fleet-webhook.example.com/webhook`
4. Set **Content type**: `application/json`
5. Set **Secret**: (save this for the next step)
6. Select events: **Just the push event**

### Step 3: Create the Webhook Secret in Kubernetes

```bash
# Create a secret with the GitHub webhook secret
kubectl create secret generic github-webhook-secret \
  --from-literal=token=your-webhook-secret-here \
  -n cattle-fleet-system
```

### Step 4: Configure GitRepo to Use Webhook

```yaml
# gitrepo-webhook.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/my-app
  branch: main

  # Reference the webhook secret
  webhookCommitID: ""

  targets:
    - clusterSelector: {}
```

## Configuring GitLab Webhooks

```bash
# Create GitLab webhook token secret
kubectl create secret generic gitlab-webhook-secret \
  --from-literal=token=your-gitlab-token \
  -n cattle-fleet-system
```

In GitLab:
1. Navigate to **Repository > Settings > Webhooks**
2. Set URL: `https://fleet-webhook.example.com/webhook`
3. Set Secret Token
4. Check **Push events**
5. Click **Add webhook**

## Forcing an Immediate Sync

When you need Fleet to re-sync without waiting for the polling interval:

```bash
# Force immediate re-sync by clearing the tracked commit
kubectl patch gitrepo my-app -n fleet-default \
  --type=merge \
  -p '{"spec":{"revision":""}}'

# Or annotate the GitRepo to trigger a sync
kubectl annotate gitrepo my-app \
  -n fleet-default \
  fleet.cattle.io/commit="" \
  --overwrite
```

## Update Strategies

### Pause Updates for Maintenance

```bash
# Pause updates by setting a specific commit to freeze at
CURRENT_COMMIT=$(kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.status.commit}')

# Pin to the current commit (no more updates)
kubectl patch gitrepo my-app -n fleet-default \
  --type=merge \
  -p "{\"spec\":{\"revision\":\"${CURRENT_COMMIT}\"}}"

# Resume auto-updates by tracking the branch again
kubectl patch gitrepo my-app -n fleet-default \
  --type=merge \
  -p '{"spec":{"revision":"","branch":"main"}}'
```

## Monitoring Auto-Update Activity

```bash
# Watch Fleet sync in real-time
kubectl get gitrepo -n fleet-default -w

# Check the last synced commit
kubectl get gitrepo my-app -n fleet-default \
  -o jsonpath='{.status.commit}'

# View sync history via events
kubectl get events -n fleet-default \
  --sort-by='.lastTimestamp' \
  | grep my-app

# Check bundle update times
kubectl get bundles -n fleet-default \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.metadata.creationTimestamp}{"\n"}{end}'
```

## Troubleshooting Auto-Update Issues

```bash
# Check if Fleet can reach the Git repository
kubectl describe gitrepo my-app -n fleet-default | grep -A 5 "Conditions:"

# Check the Fleet gitjob controller logs
kubectl logs -n cattle-fleet-system \
  -l app=gitjob \
  --tail=50

# Verify webhook is being received
kubectl logs -n cattle-fleet-system \
  -l app=gitjob \
  | grep webhook
```

## Conclusion

Fleet's auto-update capabilities make it a true GitOps system - changes in Git automatically flow to your clusters with no manual intervention required. The combination of polling and webhook support gives you flexibility to balance between simplicity (polling) and responsiveness (webhooks). By properly configuring update intervals and leveraging webhooks for production environments, you can ensure your clusters stay synchronized with your Git repository with minimal latency while maintaining control over the update process.
