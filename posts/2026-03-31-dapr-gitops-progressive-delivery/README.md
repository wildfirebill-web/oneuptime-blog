# How to Use Dapr with GitOps and Progressive Delivery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, GitOps, Progressive Delivery, Argo CD, Flagger, Canary

Description: Implement GitOps for Dapr component management using Argo CD and progressive delivery using Flagger for canary deployments of Dapr-enabled services.

---

## GitOps for Dapr Components

GitOps treats your Git repository as the single source of truth for infrastructure state. For Dapr, this means all component YAML files, resiliency policies, configurations, and subscriptions are stored in Git and applied automatically by a GitOps controller.

## Repository Structure for GitOps

Organize your GitOps repository:

```bash
gitops-repo/
├── clusters/
│   ├── staging/
│   │   └── dapr/
│   │       ├── components/
│   │       │   ├── statestore.yaml
│   │       │   └── pubsub.yaml
│   │       └── config/
│   │           ├── tracing.yaml
│   │           └── resiliency.yaml
│   └── production/
│       └── dapr/
│           ├── components/
│           └── config/
└── apps/
    ├── order-service/
    └── inventory-service/
```

## Setting Up Argo CD for Dapr Components

Install Argo CD and create an Application that syncs Dapr components:

```bash
# Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Create an Argo CD Application for Dapr components:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dapr-components-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:myorg/gitops-repo.git
    targetRevision: main
    path: clusters/production/dapr
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

When you push a component change to Git, Argo CD automatically applies it.

## Progressive Delivery with Flagger

Use Flagger to automate canary deployments of Dapr-enabled services:

```bash
# Install Flagger
helm install flagger flagger/flagger \
  --namespace flagger-system \
  --create-namespace \
  --set meshProvider=nginx \
  --set metricsServer=http://prometheus:9090
```

## Flagger Canary for Dapr Service

Create a Canary resource for a Dapr-enabled service:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: order-processor
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  service:
    port: 8080
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        min: 99
        interval: 1m
      - name: request-duration
        max: 500           # Max 500ms P99 latency
        interval: 1m
    webhooks:
      - name: dapr-health-check
        url: http://flagger-loadtester/
        metadata:
          type: cmd
          cmd: "curl -sf http://order-processor-canary.production:3500/v1.0/healthz"
```

## Dapr-Specific Canary Metrics

Add Dapr-specific success metrics to the Flagger analysis:

```yaml
metrics:
  - name: dapr-service-invocation-success
    templateRef:
      name: dapr-invocation-success-rate
      namespace: flagger-system
    min: 99
    interval: 1m
```

Create the metric template:

```yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: dapr-invocation-success-rate
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus:9090
  query: |
    100 - (
      sum(rate(dapr_service_invocation_req_sent_total{app_id="{{ target }}",success="false"}[1m]))
      /
      sum(rate(dapr_service_invocation_req_sent_total{app_id="{{ target }}"}[1m]))
    ) * 100
```

## GitOps Workflow for Component Updates

When a team needs to change a Dapr component (e.g., increase Redis TTL):

```bash
# 1. Create a branch
git checkout -b update-redis-ttl

# 2. Edit the component file
vim clusters/production/dapr/components/statestore.yaml

# 3. Open a pull request - team reviews the change
git push origin update-redis-ttl
gh pr create --title "Increase Redis TTL to 7 days"

# 4. After approval and merge, Argo CD applies automatically
# Check sync status
argocd app sync dapr-components-production
argocd app wait dapr-components-production --health
```

## Summary

GitOps with Argo CD provides automatic, auditable synchronization of Dapr component configurations from Git, ensuring the cluster state always matches the repository. Flagger progressive delivery enables safe canary deployments of Dapr services by incrementally shifting traffic and analyzing Dapr-specific success metrics, automatically rolling back if error rates or latency exceed defined thresholds.
