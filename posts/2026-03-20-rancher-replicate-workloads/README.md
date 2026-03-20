# How to Replicate Workloads Across Rancher Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Multi-Cluster, Replication, Fleet

Description: Learn how to replicate Kubernetes workloads across multiple Rancher-managed clusters for high availability, disaster recovery, and geo-distribution.

## Introduction

Replicating workloads across clusters is essential for high availability (survive a cluster failure), disaster recovery (recover quickly after a region outage), and geographic distribution (serve users from the nearest region). Rancher Fleet, combined with Kustomize overlays and automated image tag propagation, makes multi-cluster workload replication straightforward.

## Replication Strategies

| Strategy | Use Case | Mechanism |
|---|---|---|
| **Active-Active** | HA / geo-distribution | Same workload in all clusters with traffic routing |
| **Active-Passive** | Disaster Recovery | Primary cluster serves traffic; secondary is on standby |
| **GitOps Sync** | Config consistency | Fleet ensures identical manifests on all clusters |

## Step 1: Structure the GitOps Repository

```text
workload-replication/
├── base/
│   ├── deployment.yaml      # Base deployment spec
│   ├── service.yaml
│   ├── hpa.yaml
│   └── kustomization.yaml
├── clusters/
│   ├── us-east-1/
│   │   ├── kustomization.yaml
│   │   ├── patch-replicas.yaml   # 5 replicas for primary
│   │   └── fleet.yaml
│   ├── eu-west-1/
│   │   ├── kustomization.yaml
│   │   ├── patch-replicas.yaml   # 3 replicas for EU
│   │   └── fleet.yaml
│   └── ap-southeast-1/
│       ├── kustomization.yaml
│       ├── patch-replicas.yaml   # 2 replicas for APAC
│       └── fleet.yaml
```

## Step 2: Create the Base Workload

```yaml
# base/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3   # Overridden per cluster
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: ghcr.io/my-org/myapp:1.0.0   # Image tag updated by CI/CD
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 5
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

## Step 3: Create Per-Cluster Overlays

```yaml
# clusters/us-east-1/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patches:
  - path: patch-replicas.yaml
commonLabels:
  cluster-region: us-east-1
```

```yaml
# clusters/us-east-1/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 5
```

```yaml
# clusters/us-east-1/fleet.yaml
defaultNamespace: production
kustomize:
  dir: .
targets:
  - name: us-east-primary
    clusterSelector:
      matchLabels:
        region: us-east-1
        environment: production
```

## Step 4: Register Fleet GitRepos for Each Region

```yaml
# fleet-gitrepos.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp-us-east
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/workload-replication
  branch: main
  paths:
    - clusters/us-east-1
  targets:
    - clusterSelector:
        matchLabels:
          region: us-east-1
---
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp-eu-west
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/workload-replication
  branch: main
  paths:
    - clusters/eu-west-1
  targets:
    - clusterSelector:
        matchLabels:
          region: eu-west-1
```

## Step 5: Automate Image Tag Propagation

When a new version is released, update the image tag in the base manifest:

```yaml
# .github/workflows/replicate-update.yaml
name: Replicate Workload Update

on:
  workflow_dispatch:
    inputs:
      image-tag:
        description: 'Image tag to deploy'
        required: true

jobs:
  update-all-clusters:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: my-org/workload-replication
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update image tag in base
        run: |
          cd base
          kustomize edit set image \
            myapp=ghcr.io/my-org/myapp:${{ github.event.inputs.image-tag }}

      - name: Commit and push
        run: |
          git config user.email "ci@example.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "chore: replicate myapp ${{ github.event.inputs.image-tag }} to all clusters"
          git push
```

## Step 6: Implement Active-Passive Failover

```yaml
# For active-passive, use Fleet's pause feature on the passive cluster
# Active cluster: normal GitRepo
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp-primary
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/workload-replication
  branch: main
  paused: false   # Active
  targets:
    - clusterSelector:
        matchLabels:
          role: primary
---
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: myapp-secondary
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/workload-replication
  branch: main
  paused: true    # Standby - un-pause to promote
  targets:
    - clusterSelector:
        matchLabels:
          role: secondary
```

```bash
# Failover script - promote secondary to active
kubectl patch gitrepo myapp-secondary -n fleet-default \
  --type=merge \
  -p '{"spec":{"paused": false}}'

# Update DNS/load balancer to route to secondary cluster
# (Use Route 53 health checks, Azure Traffic Manager, or Google Cloud DNS for automated failover)
```

## Step 7: Verify Replication Status

```bash
# Check replication status across all clusters
kubectl get bundledeployment -A \
  -o custom-columns="CLUSTER:.spec.clusterName,NAME:.metadata.name,READY:.status.ready,STATE:.status.state"

# Compare deployed versions
for cluster in $(kubectl get bundledeployment -A -o jsonpath='{range .items[*]}{.spec.clusterName}{"\n"}{end}' | sort -u); do
  echo -n "${cluster}: "
  kubectl get bundledeployment -A \
    --field-selector spec.clusterName="${cluster}" \
    -o jsonpath='{.items[0].spec.options.helm.values.image.tag}' 2>/dev/null \
    || echo "unknown"
done
```

## Conclusion

Replicating workloads across Rancher-managed clusters combines Fleet's GitOps synchronization with Kustomize overlays for per-cluster customization. For high availability, maintain identical workloads across multiple active clusters with load balancer routing. For disaster recovery, use Fleet's pause feature to pre-deploy standby workloads that can be activated in minutes. Automating image tag propagation through CI/CD ensures consistent, simultaneous updates across all cluster regions.
