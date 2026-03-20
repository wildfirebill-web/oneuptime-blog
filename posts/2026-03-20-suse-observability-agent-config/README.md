# How to Configure the SUSE Observability Agent

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SUSE Observability, Agent Configuration, Kubernetes, Monitoring, Helm, SUSE Rancher, Topology

Description: Learn how to configure the SUSE Observability agent for Kubernetes clusters including collector settings, custom tags, log collection, and integration with specific workload types.

---

The SUSE Observability agent runs on each Kubernetes cluster you want to monitor. Proper configuration ensures all topology data, metrics, logs, and events flow correctly to the SUSE Observability server.

---

## Agent Architecture

```
┌─────────────────────────────────────────┐
│          Kubernetes Cluster             │
│                                         │
│  ┌─────────────┐   ┌─────────────────┐  │
│  │ Node Agent  │   │  Cluster Agent  │  │
│  │ (DaemonSet) │   │  (Deployment)   │  │
│  └──────┬──────┘   └────────┬────────┘  │
│         │                   │           │
│         └─────────┬─────────┘           │
│                   │                     │
└───────────────────┼─────────────────────┘
                    │
              SUSE Observability
                  Server
```

---

## Step 1: Basic Agent Configuration

```yaml
# agent-values.yaml
stackstate:
  apiKey: "your-api-key"
  cluster:
    name: "production-us-west"    # Unique name for this cluster
    authToken: ""                 # Optional: for secure cluster identification
  url: "https://observability.example.com/receiver/solarwinds"

# Node agent runs on every node
nodeAgent:
  enabled: true
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule

# Cluster agent collects cluster-level resources
clusterAgent:
  enabled: true
  collection:
    kubernetesTopology: true
    kubernetesMetrics: true
    kubernetesEvents: true
    kubernetesState: true
```

---

## Step 2: Configure Log Collection

```yaml
# Enable log collection from pods
logsAgent:
  enabled: true
  # Collect logs from all containers by default
  containerCollectAll: true
  # Exclude specific namespaces from log collection
  excludeNamespaces:
    - kube-system
    - cert-manager

# Add log processing rules
logProcessing:
  rules:
    - name: mask-sensitive-data
      type: mask_sequences
      pattern: \d{4}-\d{4}-\d{4}-\d{4}    # Mask credit card patterns
      replacePlaceholder: "[MASKED]"
```

---

## Step 3: Add Custom Tags to All Data

Tags help filter and group data in the Observability UI:

```yaml
# Apply custom tags to all collected data
global:
  extraEnvVars:
    - name: STS_TAGS
      value: "env:production,team:platform,region:us-west-2"

# Or configure per-component
nodeAgent:
  extraEnvVars:
    - name: STS_TAGS
      value: "component:node-agent"
```

---

## Step 4: Configure Checks for Specific Integrations

The agent supports built-in checks for common services:

```yaml
# agent-values.yaml
nodeAgent:
  config:
    override:
      - name: auto_conf/nginx.yaml
        path: /etc/stackstate-agent/conf.d
        data: |
          ad_identifiers:
            - nginx
          init_config:
          instances:
            - nginx_status_url: http://%%host%%/nginx_status
```

---

## Step 5: Configure Resource Limits

```yaml
# Set resource requests and limits for the agent
nodeAgent:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

clusterAgent:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi
```

---

## Step 6: Apply the Configuration

```bash
# Upgrade the agent deployment with new values
helm upgrade suse-observability-agent \
  suse-observability-agent/suse-observability-agent \
  --namespace suse-observability \
  --values agent-values.yaml \
  --wait

# Verify pods restarted with new config
kubectl get pods -n suse-observability

# Check agent logs for configuration errors
kubectl logs -n suse-observability daemonset/suse-observability-agent-node-agent | head -50
```

---

## Step 7: Verify the Agent is Reporting

```bash
# Check if the agent is sending data
kubectl logs -n suse-observability deployment/suse-observability-agent-cluster-agent \
  | grep -i "topology\|metrics\|error"

# List all running checks
kubectl exec -n suse-observability \
  $(kubectl get pod -n suse-observability -l app.kubernetes.io/component=node-agent -o name | head -1) \
  -- stackstate-agent check kubernetes_state
```

---

## Troubleshooting Agent Issues

```bash
# Agent not connecting to server
kubectl logs -n suse-observability daemonset/suse-observability-agent-node-agent \
  | grep -i "connection\|refused\|timeout"

# Check the API key is correct
kubectl get secret -n suse-observability suse-observability-agent \
  -o jsonpath='{.data.stackstate-api-key}' | base64 -d

# Restart the agent
kubectl rollout restart daemonset/suse-observability-agent-node-agent -n suse-observability
```

---

## Best Practices

- Set a meaningful `cluster.name` that identifies the environment and region — this name appears in the topology view and cannot be easily changed.
- Enable log collection selectively for namespaces with high log volume to avoid overwhelming the Observability server.
- Use `tolerations` on the node agent to ensure it runs on control-plane nodes for complete cluster visibility.
