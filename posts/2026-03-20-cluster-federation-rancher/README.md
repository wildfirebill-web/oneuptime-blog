# How to Implement Cluster Federation with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Cluster Federation, Multi-Cluster, Kubernetes, Fleet, Submariner, High Availability

Description: Learn how to implement cluster federation in Rancher to synchronize workloads, policies, and namespaces across multiple Kubernetes clusters for high availability and geographic distribution.

---

Cluster federation in Rancher allows you to treat multiple clusters as a single logical unit, distributing workloads, sharing policies, and enabling cross-cluster service connectivity.

---

## Federation Approaches in Rancher

Rancher supports several patterns for cluster federation:

| Approach | Tool | Best For |
|---|---|---|
| GitOps-based federation | Rancher Fleet | Config and app distribution |
| Cross-cluster networking | Submariner | Pod/service communication |
| Policy federation | Rancher Projects | Namespace and RBAC templates |

---

## Approach 1: Rancher Fleet for Config Federation

Deploy the same application configuration to multiple clusters using a single `GitRepo`:

```yaml
# gitrepo-federated.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: federated-apps
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/k8s-configs.git
  branch: main
  paths:
    - apps/

  # Deploy to ALL clusters with env=production label
  targets:
    - name: all-production
      clusterSelector:
        matchLabels:
          env: production
      # Override values per region
      helm:
        values:
          region: "{{ .ClusterLabels.region }}"
```

---

## Approach 2: Submariner for Cross-Cluster Networking

Submariner creates encrypted tunnels between clusters enabling pod-to-pod and service communication:

```bash
# Install the subctl CLI
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:~/.local/bin

# Deploy the broker on the primary cluster
subctl deploy-broker --kubeconfig cluster1.yaml

# Join each cluster to the broker
subctl join broker-info.subm \
  --kubeconfig cluster1.yaml \
  --clusterid cluster1 \
  --natt=false

subctl join broker-info.subm \
  --kubeconfig cluster2.yaml \
  --clusterid cluster2 \
  --natt=false
```

After joining, export a service so it is discoverable from other clusters:

```bash
# Export the my-app service from cluster1 to all federated clusters
subctl export service my-app \
  --namespace my-app \
  --kubeconfig cluster1.yaml
```

From cluster2, access the service using its federated DNS name:

```bash
# Access service across clusters via Submariner DNS
curl http://my-app.my-app.svc.clusterset.local
```

---

## Approach 3: Rancher Projects for Policy Federation

Rancher Projects allow you to template namespaces and resource quotas and apply them across clusters consistently:

```yaml
# project-template.yaml (applied via Rancher management API)
apiVersion: management.cattle.io/v3
kind: ProjectTemplate
metadata:
  name: standard-team-project
spec:
  resourceQuota:
    limit:
      pods: "50"
      requestsCpu: "10"
      requestsMemory: 20Gi
  namespaceDefaultResourceQuota:
    limit:
      requestsCpu: "2"
      requestsMemory: 4Gi
```

---

## Step 3: Federated Health Monitoring

Use Rancher's multi-cluster dashboard or Grafana to monitor all federated clusters. Create a Grafana dashboard variable for the `cluster` label:

```promql
# Alert when any cluster in the federation has high error rate
sum by (cluster) (
  rate(http_requests_total{status=~"5.."}[5m])
)
/ sum by (cluster) (
  rate(http_requests_total[5m])
) > 0.01
```

---

## Best Practices

- Treat each cluster as an autonomous unit — federation should augment, not couple clusters tightly.
- Use Fleet's **per-cluster value overrides** to customize behavior (replicas, resource limits) per cluster.
- Test cross-cluster failover regularly to ensure Submariner tunnels remain healthy.
- Apply federated RBAC via Rancher role templates to avoid per-cluster permission drift.
