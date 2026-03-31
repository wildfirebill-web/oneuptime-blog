# How to Manage K3s Edge Clusters Remotely

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Edge Computing, Remote Management, Rancher, Fleet, GitOps, SUSE Rancher

Description: Learn how to remotely manage K3s edge clusters using Rancher, Fleet GitOps, and secure tunneling to deploy workloads, apply configurations, and monitor cluster health from a central location.

---

Managing K3s clusters at the edge requires secure remote access, centralized configuration management, and automated deployment pipelines that work even over unreliable connections.

---

## Architecture

```text
┌─────────────────────────────────────┐
│         Central Rancher             │
│                                     │
│  ┌─────────┐   ┌──────────────┐    │
│  │ Rancher │   │ Fleet GitOps │    │
│  │   UI    │   │   Manager    │    │
│  └────┬────┘   └──────┬───────┘    │
│       │               │            │
└───────┼───────────────┼────────────┘
        │               │
   Secure Tunnel   Git Sync
        │               │
┌───────┼───────────────┼────────────┐
│       │   Edge Site   │            │
│  ┌────▼────┐   ┌──────▼───────┐   │
│  │ K3s     │   │ Fleet Agent  │   │
│  │ Cluster │   │              │   │
│  └─────────┘   └──────────────┘   │
└────────────────────────────────────┘
```

---

## Step 1: Register Edge K3s Clusters with Rancher

On the Rancher management server, create an import command for the edge cluster:

```bash
# In Rancher UI: Cluster Management → Import Existing → Generic

# Run the generated command on the edge cluster

kubectl apply -f https://rancher.example.com/v3/import/<token>.yaml

# Verify the cluster appears in Rancher
kubectl get cluster -n fleet-default
```

---

## Step 2: Deploy Fleet Agent for GitOps

Fleet agents handle GitOps-based workload deployment even when the connection to Rancher is intermittent:

```bash
# The Fleet agent is automatically installed when you import a cluster into Rancher
# Verify it's running
kubectl get pods -n cattle-fleet-system

# Check Fleet agent status
kubectl describe bundle -n fleet-local
```

---

## Step 3: Deploy Workloads via Fleet (Works Offline)

```yaml
# gitrepo-edge.yaml (applied on the Rancher management cluster)
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: edge-apps
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/edge-manifests
  branch: main
  paths:
    - apps/

  # Target edge clusters by label
  targets:
    - name: edge-sites
      clusterSelector:
        matchLabels:
          cluster-type: edge
```

Fleet agents pull updates from Git periodically, so deployments work even when the edge cluster temporarily loses connectivity to Rancher.

---

## Step 4: Access Edge Cluster via kubectl through Rancher Proxy

```bash
# Download the edge cluster kubeconfig from Rancher
# Rancher UI → Cluster → Download KubeConfig

# Or use the Rancher CLI
rancher context switch edge-cluster-1
rancher kubectl get pods -A
```

---

## Step 5: Set Up Automated Health Monitoring

```yaml
# Edge clusters report health metrics back to Rancher
# Configure alerts for offline clusters in Rancher:

# monitoring-alert.yaml (alertmanager rule)
- alert: EdgeClusterOffline
  expr: rancher_cluster_condition_ready{condition="Ready"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Edge cluster {{ $labels.cluster_name }} is offline"
```

---

## Step 6: Configure Automatic Reconnection

K3s agents are configured to automatically reconnect to the server. Ensure the agent service is configured for resilience:

```bash
# On the edge K3s node
cat /etc/systemd/system/k3s-agent.service.d/override.conf

# Ensure Restart=always and a retry delay
[Service]
Restart=always
RestartSec=10s
```

---

## Step 7: Manage Edge Cluster Updates

```bash
# Update K3s version on edge clusters using Fleet
# fleet.yaml in the edge manifests repo

# Or use Rancher's automated upgrade controller
kubectl apply -f - <<EOF
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-server-upgrade
  namespace: system-upgrade
spec:
  concurrency: 1
  channel: https://update.k3s.io/v1-release/channels/stable
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/control-plane: "true"
  upgrade:
    image: rancher/k3s-upgrade
EOF
```

---

## Step 8: Handle Configuration Drift

```bash
# Check Fleet status across all edge clusters
kubectl get gitrepo -n fleet-default

# Force resync if drift is detected
kubectl patch gitrepo edge-apps -n fleet-default \
  --type merge \
  -p '{"spec":{"forceSyncGeneration":1}}'
```

---

## Best Practices

- Use Fleet for workload deployment to edge clusters - it works asynchronously and tolerates intermittent connectivity better than direct kubectl applies.
- Label edge clusters consistently (`cluster-type: edge`, `region: store-42`) to enable precise Fleet targeting without hardcoding cluster names.
- Store kubeconfigs for edge clusters in a secure vault - if Rancher is unavailable, you can still access edge clusters directly using the kubeconfig.
