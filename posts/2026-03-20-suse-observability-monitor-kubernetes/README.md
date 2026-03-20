# How to Monitor Kubernetes Clusters with SUSE Observability

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SUSE Observability, Kubernetes, Monitoring, Topology, Observability

Description: Set up comprehensive Kubernetes cluster monitoring with SUSE Observability for topology, metrics, and events.

## Introduction

SUSE Observability (formerly StackState) is a full-stack observability platform that provides topology-based monitoring for Kubernetes and cloud-native applications. Unlike traditional metrics-only monitoring tools, SUSE Observability builds a real-time topology map of your infrastructure, making it easier to understand dependencies and pinpoint root causes. This guide covers How to Monitor Kubernetes Clusters with SUSE Observability.

## Prerequisites

- Kubernetes cluster (v1.24+)
- Rancher v2.7+ (for Rancher-integrated installation)
- Helm v3.x
- At least 16 GB RAM available in the cluster
- A SUSE Observability license key

## Understanding SUSE Observability Architecture

SUSE Observability consists of:

- **StackState Server**: Core processing and storage engine
- **Agents**: Collect data from Kubernetes nodes and services
- **Receiver**: Accepts data from agents and integrations
- **UI**: Web interface for topology visualization and monitoring

## Step 1: Install SUSE Observability

```bash
# Add the SUSE Observability Helm repository
helm repo add suse-observability https://charts.rancher.com/server-charts/prime/suse-observability
helm repo update

# Create namespace
kubectl create namespace suse-observability

# Install with your license key
helm install suse-observability suse-observability/suse-observability \
  --namespace suse-observability \
  --set license="your-license-key" \
  --set baseUrl="https://observability.example.com"
```

## Step 2: Deploy the Agent

```bash
# Add agent Helm repository
helm repo add suse-observability-agent https://charts.rancher.com/server-charts/prime/suse-observability-agent
helm repo update

# Deploy agent on target cluster
helm install stackstate-agent suse-observability-agent/suse-observability-agent \
  --namespace stackstate \
  --create-namespace \
  --set stackstate.apiKey="your-api-key" \
  --set stackstate.url="https://observability.example.com/receiver/sinks/generic"
```

## Step 3: Configure Data Collection

```yaml
# agent-values.yaml
stackstate:
  url: "https://observability.example.com/receiver/sinks/generic"
  apiKey: "your-api-key"

# Cluster name for identification
clusterName: "production-cluster"

# Enable all Kubernetes data collection
kubernetes:
  enabled: true
  
# Collect container metrics
containerRuntime:
  enabled: true
  
# Collect node metrics
nodeAgent:
  enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
```

```bash
helm upgrade --install stackstate-agent \
  suse-observability-agent/suse-observability-agent \
  --namespace stackstate \
  --values agent-values.yaml
```

## Step 4: Verify Data Collection

```bash
# Check agent pods are running
kubectl get pods -n stackstate

# View agent logs
kubectl logs -n stackstate \
  -l app.kubernetes.io/name=stackstate-agent \
  --follow

# Check agent is sending data
kubectl logs -n stackstate \
  -l app.kubernetes.io/name=stackstate-agent \
  | grep "Successfully sent"
```

## Step 5: Access the SUSE Observability UI

```bash
# Get the service URL
kubectl get svc -n suse-observability \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'

# Or set up port forwarding
kubectl port-forward -n suse-observability \
  svc/suse-observability 8080:8080 &

# Access the UI
open http://localhost:8080
```

## Key Features to Configure

### Topology Maps

Navigate to **Topology** in the UI to see:
- Service dependencies
- Infrastructure relationships
- Real-time component health
- Change tracking

### Health States

Configure health propagation rules to reflect component status:

```yaml
# Example health rule configuration via API
health_rule:
  name: "High CPU Usage"
  component_type: "kubernetes-pod"
  conditions:
    - metric: "cpu.usage.percent"
      threshold: 80
      operator: ">"
  severity: "warning"
```

### Monitors and Alerts

```bash
# Configure alert channels via CLI
stackstate monitor create \
  --name "Pod Restart Alert" \
  --query "kubernetes_pod_restart_count > 5" \
  --severity warning \
  --notify-channel slack
```

## Troubleshooting

```bash
# Check server health
kubectl exec -n suse-observability \
  -it $(kubectl get pods -n suse-observability -l app=suse-observability -o name | head -1) \
  -- curl -s localhost:7070/api/server/health | jq .

# Restart agent if not sending data
kubectl rollout restart daemonset stackstate-agent -n stackstate

# Check agent configuration
kubectl get configmap stackstate-agent-config -n stackstate -o yaml
```

## Conclusion

How to Monitor Kubernetes Clusters with SUSE Observability enables comprehensive observability of your Kubernetes infrastructure through topology-based monitoring. SUSE Observability's unique approach of building a real-time dependency map makes it significantly easier to understand the impact of changes and identify root causes of issues. By combining topology visualization with metrics, events, and traces, it provides the full context needed for effective troubleshooting.
