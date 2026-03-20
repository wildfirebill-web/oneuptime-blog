# How to Configure Rancher Agent Resource Allocation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Rancher Agent, Resource Management, Kubernetes, Cattle Agent, Performance

Description: Configure resource requests and limits for Rancher Agent pods to ensure stable cluster management without impacting application workloads.

## Introduction

Rancher deploys several agent components in managed downstream clusters: `cattle-cluster-agent`, `cattle-node-agent`, and Fleet agents. By default, these have minimal resource configurations. Properly tuning these ensures stable management communication without starving application workloads.

## Rancher Agent Components

| Component | Location | Purpose |
|---|---|---|
| cattle-cluster-agent | cattle-system namespace | Manages cluster state, syncs with Rancher Server |
| cattle-node-agent | DaemonSet, all nodes | Node-level operations, catalog deployment |
| fleet-agent | fleet-system namespace | GitOps and application deployment |

## Step 1: Configure cattle-cluster-agent Resources

The cluster agent is the most resource-intensive agent. Configure it based on cluster size:

```yaml
# Patch the cluster agent deployment resources

kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type=merge \
  -p '{
    "spec": {
      "template": {
        "spec": {
          "containers": [{
            "name": "cluster-register",
            "resources": {
              "requests": {
                "cpu": "200m",
                "memory": "256Mi"
              },
              "limits": {
                "cpu": "1000m",
                "memory": "1Gi"
              }
            }
          }]
        }
      }
    }
  }'
```

## Step 2: Configure Agent Resources via Rancher Server

For new clusters, configure agent resources in the cluster spec:

```yaml
# Rancher cluster spec
agentEnvVars:
  - name: CATTLE_AGENT_LIVENESS_TIMEOUT
    value: "120"

# Agent resource settings (in Rancher 2.7+)
cattle-cluster-agent:
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2000m"
      memory: "2Gi"
```

## Step 3: Configure cattle-node-agent Resources

Node agents run on every node. Keep their footprint small:

```bash
kubectl patch daemonset cattle-node-agent \
  -n cattle-system \
  --type=merge \
  -p '{
    "spec": {
      "template": {
        "spec": {
          "containers": [{
            "name": "agent",
            "resources": {
              "requests": {
                "cpu": "50m",
                "memory": "64Mi"
              },
              "limits": {
                "cpu": "250m",
                "memory": "256Mi"
              }
            }
          }]
        }
      }
    }
  }'
```

## Step 4: Assign Agents to Specific Nodes

In large clusters, pin agent pods to control plane nodes to avoid competing with application workloads:

```bash
kubectl patch deployment cattle-cluster-agent \
  -n cattle-system \
  --type=merge \
  -p '{
    "spec": {
      "template": {
        "spec": {
          "nodeSelector": {
            "node-role.kubernetes.io/control-plane": "true"
          }
        }
      }
    }
  }'
```

## Step 5: Monitor Agent Resource Usage

```bash
# Check agent resource consumption
kubectl top pods -n cattle-system
kubectl top pods -n fleet-system

# Alert if cluster agent memory exceeds 80% of limit
# (indicates need to increase limits)
```

## Conclusion

Properly sized Rancher Agent resources ensure stable cluster management without impacting application performance. For clusters with 100+ nodes or 500+ workloads, increase cluster agent limits to 2 CPU / 4GB RAM to handle the higher volume of state synchronization events.
