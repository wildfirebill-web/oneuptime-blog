# How to Install Portainer Agent on Kubernetes via Helm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, Agent, DevOps

Description: Deploy the Portainer Agent on Kubernetes using Helm charts for centralized cluster management from a Portainer server.

---

The Portainer Agent for Kubernetes runs as a DaemonSet, deploying a pod on each node to enable volume browsing and other per-node features. Helm is the recommended installation method.

## Prerequisites

- `helm` v3 installed
- `kubectl` configured for the target cluster
- Portainer server accessible from the cluster

## Add the Portainer Helm Repository

```bash
# Add the official Portainer Helm repository
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

# View available charts
helm search repo portainer
```

## Install the Portainer Agent

### Standard Agent (for Portainer Agent-based connection)

```bash
# Create the portainer namespace
kubectl create namespace portainer

# Install the Portainer Agent
helm install portainer-agent portainer/portainer-agent \
  -n portainer \
  --set env.serverAddress="tcp://portainer.example.com:9001"
```

### Edge Agent (for Edge environments)

```bash
# Install the Edge Agent variant
helm install portainer-agent portainer/portainer-agent \
  -n portainer \
  --create-namespace \
  --set env.edge.enable=true \
  --set env.edge.id="YOUR-EDGE-ID" \
  --set env.edge.key="YOUR-EDGE-KEY" \
  --set env.edge.insecure=false
```

## Verify the Installation

```bash
# Check agent pods are running on all nodes
kubectl get pods -n portainer -l app=portainer-agent

# Check the DaemonSet
kubectl get daemonset -n portainer

# View agent logs
kubectl logs -n portainer -l app=portainer-agent --tail=20
```

## Configure Values

Create a `values.yaml` for custom configuration:

```yaml
# values.yaml for Portainer Agent
image:
  repository: portainer/agent
  tag: latest

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"

tolerations:
  # Allow running on master/control-plane nodes
  - key: node-role.kubernetes.io/master
    effect: NoSchedule
  - key: node-role.kubernetes.io/control-plane
    effect: NoSchedule
```

```bash
# Install with custom values
helm install portainer-agent portainer/portainer-agent \
  -n portainer \
  -f values.yaml
```

## Add the Kubernetes Environment to Portainer

After agent installation, add the environment to Portainer:

1. In Portainer, navigate to **Environments > Add environment**
2. Select **Agent**
3. Enter the agent URL: `tcp://<node-ip>:9001`
4. Or use the Kubernetes service DNS if Portainer is in the same cluster

---

*Monitor your Kubernetes workloads with [OneUptime](https://oneuptime.com) after connecting to Portainer.*
