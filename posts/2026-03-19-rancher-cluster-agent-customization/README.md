# How to Configure Cluster Agent Customization in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Templates

Description: Learn how to customize Rancher cluster and node agent configurations for resource management, tolerations, and scheduling.

Rancher deploys cluster agents and node agents to managed clusters for communication and management. Customizing these agents allows you to control their resource usage, scheduling, and behavior. This guide covers how to configure agent customization options in Rancher.

## Prerequisites

- Rancher v2.6 or later
- Admin access to Rancher
- A managed Kubernetes cluster
- Familiarity with Kubernetes resource requests, limits, and scheduling concepts

## Understanding Rancher Agents

Rancher uses two types of agents on managed clusters:

- **Cluster Agent**: A Deployment that handles communication between the Rancher server and the downstream cluster. It manages cluster-level operations.
- **Node Agent**: A DaemonSet that runs on every node and handles node-level operations like executing commands and managing node resources.

## Step 1: Access Agent Customization

Access the agent settings during cluster creation or editing:

1. Navigate to **Cluster Management**.
2. Click **Create** for a new cluster or select an existing cluster and click **Edit Config**.
3. Scroll to the **Agent Environment Variables** or **Cluster Agent** section.
4. Look for the **Agent Customization** options.

## Step 2: Configure Cluster Agent Resources

Set resource requests and limits for the cluster agent:

```yaml
# Cluster agent resource configuration

apiVersion: apps/v1
kind: Deployment
metadata:
  name: cattle-cluster-agent
  namespace: cattle-system
spec:
  template:
    spec:
      containers:
        - name: cluster-register
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

In the Rancher UI, set these values under the agent customization section:

```plaintext
Cluster Agent CPU Request: 200m
Cluster Agent CPU Limit: 500m
Cluster Agent Memory Request: 256Mi
Cluster Agent Memory Limit: 512Mi
```

## Step 3: Configure Node Agent Resources

Set resource requests and limits for the node agent DaemonSet:

```yaml
# Node agent resource configuration
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cattle-node-agent
  namespace: cattle-system
spec:
  template:
    spec:
      containers:
        - name: agent
          resources:
            requests:
              cpu: 50m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

In the UI:

```plaintext
Node Agent CPU Request: 50m
Node Agent CPU Limit: 200m
Node Agent Memory Request: 128Mi
Node Agent Memory Limit: 256Mi
```

## Step 4: Add Agent Tolerations

Configure tolerations so agents can run on tainted nodes:

```yaml
# Cluster agent tolerations
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/controlplane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
        - key: dedicated
          operator: Equal
          value: infra
          effect: NoSchedule
```

In the Rancher UI:

1. Under **Cluster Agent Tolerations**, click **Add Toleration**.
2. Enter the toleration details for each taint your nodes may have.

For the node agent, add tolerations so it runs on all nodes:

```yaml
# Node agent tolerations - must tolerate all node taints
tolerations:
  - operator: Exists
```

## Step 5: Configure Agent Affinity Rules

Set node affinity for the cluster agent to control where it runs:

```yaml
# Cluster agent node affinity
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node-role.kubernetes.io/controlplane
                    operator: In
                    values:
                      - "true"
            - weight: 1
              preference:
                matchExpressions:
                  - key: node-role.kubernetes.io/worker
                    operator: In
                    values:
                      - "true"
```

This configuration prefers scheduling the cluster agent on control plane nodes but allows it to run on worker nodes if necessary.

## Step 6: Set Agent Environment Variables

Add custom environment variables to the agents:

1. In the cluster configuration, find **Agent Environment Variables**.
2. Add variables as needed.

```yaml
# Environment variables for agents
env:
  - name: CATTLE_AGENT_CONNECT_TIMEOUT
    value: "300"
  - name: CATTLE_AGENT_RETRY_TIMEOUT
    value: "120"
  - name: HTTP_PROXY
    value: "http://proxy.example.com:8080"
  - name: HTTPS_PROXY
    value: "http://proxy.example.com:8080"
  - name: NO_PROXY
    value: "localhost,127.0.0.1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local"
```

## Step 7: Configure Agent Image Overrides

Use custom agent images when running in air-gapped environments:

```bash
# Set the agent image globally
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "registry.internal.example.com/rancher/rancher-agent:v2.8.x"}' \
  "https://rancher.example.com/v3/settings/agent-image"
```

For per-cluster agent image overrides, configure the cluster spec:

```yaml
spec:
  agentImageOverride: registry.internal.example.com/rancher/rancher-agent:v2.8.x
```

## Step 8: Configure Agent Priority Classes

Set priority classes for agents to ensure they are not evicted during resource pressure:

```yaml
# Create a priority class for Rancher agents
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: rancher-agent-priority
value: 1000000
globalDefault: false
description: "Priority class for Rancher agents"
```

Apply the priority class to agents:

```yaml
spec:
  template:
    spec:
      priorityClassName: rancher-agent-priority
```

## Step 9: Troubleshoot Agent Issues

When agents are not functioning correctly, use these diagnostic steps:

```bash
# Check cluster agent status
kubectl get deployment cattle-cluster-agent -n cattle-system

# Check node agent status
kubectl get daemonset cattle-node-agent -n cattle-system

# View cluster agent logs
kubectl logs -l app=cattle-cluster-agent -n cattle-system --tail=100

# View node agent logs on a specific node
kubectl logs -l app=cattle-agent -n cattle-system --tail=100 \
  --field-selector spec.nodeName=<node-name>

# Check agent resource usage
kubectl top pod -n cattle-system

# Describe the cluster agent deployment
kubectl describe deployment cattle-cluster-agent -n cattle-system
```

Common issues and solutions:

| Issue | Cause | Solution |
|-------|-------|----------|
| Agent OOMKilled | Memory limit too low | Increase memory limit |
| Agent not scheduled | Missing tolerations | Add required tolerations |
| Agent CrashLoopBackOff | Cannot reach Rancher server | Check proxy settings and network connectivity |
| Agent pending | Insufficient resources | Reduce resource requests or add capacity |

## Step 10: Apply Changes to Existing Clusters

Update agent configuration on running clusters:

1. Go to **Cluster Management**.
2. Select the cluster.
3. Click **Edit Config**.
4. Modify the agent customization settings.
5. Click **Save**.

Rancher will perform a rolling update of the agents with the new configuration.

You can also modify agents directly with kubectl:

```bash
# Edit the cluster agent deployment
kubectl edit deployment cattle-cluster-agent -n cattle-system

# Edit the node agent daemonset
kubectl edit daemonset cattle-node-agent -n cattle-system
```

Note that direct kubectl edits may be overwritten by Rancher on the next reconciliation. Always use the Rancher UI or API for persistent changes.

## Best Practices

- **Set appropriate resource limits**: Monitor agent resource usage and set limits that prevent resource starvation without being overly restrictive.
- **Always add control plane tolerations**: The cluster agent should tolerate control plane taints to ensure it can run on control plane nodes.
- **Use priority classes**: Assign high-priority classes to agents so they are not evicted during resource pressure.
- **Configure proxy settings centrally**: If your environment uses proxies, set the proxy environment variables at the agent level.
- **Monitor agent health**: Include agent pods in your monitoring and alerting setup to catch issues early.

## Conclusion

Customizing Rancher cluster and node agents ensures they operate efficiently within your cluster's resource constraints and scheduling requirements. By configuring appropriate resource limits, tolerations, affinity rules, and environment variables, you create a robust management plane that works reliably across diverse infrastructure configurations. Take the time to tune these settings based on your specific environment, and your Rancher-managed clusters will be more stable and responsive.
