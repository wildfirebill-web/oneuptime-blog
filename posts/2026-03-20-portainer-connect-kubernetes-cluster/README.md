# How to Connect Portainer to an Existing Kubernetes Cluster - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Cluster Management, DevOps

Description: Learn how to connect an existing Kubernetes cluster to Portainer using the Portainer Agent or kubeconfig for centralized management.

## Introduction

Portainer can manage Kubernetes clusters that it's not running in. By installing the Portainer Agent on a cluster and registering it in Portainer, or by providing a kubeconfig, you can manage multiple Kubernetes clusters from a single Portainer interface. This guide covers both connection methods.

## Prerequisites

- Portainer CE or BE running (on Docker or Kubernetes)
- Target Kubernetes cluster accessible from Portainer
- kubectl access to the target cluster
- Admin access to Portainer

## Connection Methods

| Method | How it works | Best for |
|--------|-------------|---------|
| **Portainer Agent** | Agent runs in cluster, connects to Portainer | Most clusters, firewalled environments |
| **Kubeconfig** | Portainer connects directly using kubeconfig | Simple setups, clusters with direct access |
| **Edge Agent** | For clusters behind firewalls | Air-gapped, NAT environments |

## Method A: Connect via Portainer Agent

### Step 1: Add the Environment in Portainer

1. In Portainer, go to **Environments**
2. Click **+ Add environment**
3. Select **Kubernetes**
4. Select **Portainer Agent**
5. Copy the agent installation command

### Step 2: Install the Portainer Agent on the Target Cluster

```bash
# Install Portainer Agent using Helm

helm repo add portainer https://portainer.github.io/k8s/
helm repo update

helm install portainer-agent portainer/portainer-agent \
  --namespace portainer \
  --create-namespace \
  --set env.EDGE=0 \
  --set env.EDGE_KEY="your-edge-key-from-portainer"

# Or via YAML manifest provided by Portainer
kubectl apply -f https://downloads.portainer.io/ce2-21/portainer-agent-k8s-lb.yaml
```

### Step 3: Get the Agent Service IP

```bash
# Get the Portainer Agent service IP
kubectl get svc -n portainer

# For LoadBalancer:
kubectl get svc portainer-agent -n portainer -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### Step 4: Register the Environment in Portainer

1. In Portainer, fill in the **Environment details**:
   ```text
   Name:        production-k8s
   Endpoint URL: https://agent-ip:9001
   ```
2. Click **Connect**

## Method B: Connect via Kubeconfig

### Step 1: Get the Kubeconfig

```bash
# Get the cluster kubeconfig
cat ~/.kube/config

# Or export for a specific cluster
kubectl config view --minify --flatten > portainer-kubeconfig.yaml
```

### Step 2: Create a Service Account for Portainer

Instead of using admin credentials, create a dedicated service account:

```yaml
# portainer-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: portainer
  namespace: portainer
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portainer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  ref: cluster-admin    # Or create a custom role with limited permissions
subjects:
  - kind: ServiceAccount
    name: portainer
    namespace: portainer
```

```bash
kubectl apply -f portainer-sa.yaml

# Get the service account token
kubectl create token portainer \
  --namespace portainer \
  --duration=87600h    # 10 years; adjust as needed
```

### Step 3: Add Kubeconfig Environment in Portainer

1. In Portainer, go to **Environments → + Add environment**
2. Select **Kubernetes**
3. Choose **Import an existing Kubernetes environment using a kubeconfig file**
4. Paste the kubeconfig content
5. Click **Create environment**

## Method C: Edge Agent (Firewalled Clusters)

For clusters that cannot be directly reached by Portainer:

### Step 1: Add Edge Environment in Portainer

1. Go to **Environments → + Add environment**
2. Select **Kubernetes**
3. Select **Edge Agent**
4. Copy the provided edge key

### Step 2: Install Edge Agent on the Target Cluster

```bash
# Install Edge Agent (it connects OUT to Portainer, not in)
helm install portainer-edge-agent portainer/portainer-agent \
  --namespace portainer \
  --create-namespace \
  --set env.EDGE=1 \
  --set env.EDGE_KEY="your-edge-key" \
  --set env.EDGE_INSECURE_POLL=1 \
  --set env.URL="https://portainer.example.com:9443"
```

The Edge Agent initiates an outbound connection to Portainer - no inbound firewall rules needed.

## Step 4: Verify the Connection

After connecting:

1. In Portainer, go to **Environments**
2. The cluster should show as **Up** or **Connected**
3. Click on the environment to access it

```bash
# Verify agent is running
kubectl get pods -n portainer

# Check agent logs
kubectl logs -n portainer -l app=portainer-agent -f
```

## Configuring RBAC for the Connected Cluster

After connecting, configure access control in Portainer:

1. Navigate to the environment
2. Go to **Settings → Cluster**
3. Configure namespace access for teams
4. Set resource quotas per namespace (BE feature)

## Disconnecting a Cluster

To remove a cluster from Portainer:

1. In Portainer, go to **Environments**
2. Click the environment name
3. Click **Remove**

Then clean up the agent:

```bash
# Remove Portainer Agent from the cluster
helm uninstall portainer-agent --namespace portainer
kubectl delete namespace portainer
```

## Troubleshooting Connection Issues

```bash
# Test agent connectivity from Portainer host
curl -k https://agent-ip:9001/api/status

# Check agent pod logs
kubectl logs -n portainer -l app=portainer-agent

# Verify network policy allows agent communication
kubectl get networkpolicies -n portainer
```

## Conclusion

Connecting Kubernetes clusters to Portainer provides centralized management across your entire container infrastructure. Use the Portainer Agent for most scenarios, kubeconfig for simple setups, and the Edge Agent when clusters are behind firewalls. Once connected, you can manage namespaces, deployments, Helm charts, and configuration objects from a single unified interface.
