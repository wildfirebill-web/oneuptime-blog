# How to Configure Fleet Agent Options

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Agent

Description: Learn how to configure Fleet agent options to customize agent behavior, resource limits, proxy settings, and tolerations for edge and enterprise deployments.

## Introduction

The Fleet agent runs in each managed cluster and is responsible for receiving bundle deployments and applying resources. Properly configuring the agent ensures it can operate correctly in your specific environment - whether that means setting proxy configurations, resource limits, custom tolerations, or connection settings.

This guide covers all Fleet agent configuration options and how to apply them at installation time or update them for existing agents.

## Prerequisites

- Fleet installed in Rancher
- `kubectl` access to both Fleet manager and downstream clusters
- Helm v3 installed for agent installation/updates

## Fleet Agent Components

The Fleet agent consists of:
- **fleet-agent**: The main agent controller that manages bundle deployments
- **fleet-agent-bootstrap**: Initial registration component (runs once)

Both components run in the `cattle-fleet-system` namespace.

## Basic Agent Configuration at Installation

When installing the Fleet agent via Helm, you can configure all options:

```bash
# Full Fleet agent installation with options

helm install fleet-agent fleet/fleet-agent \
  --namespace cattle-fleet-system \
  --create-namespace \
  --set apiServerURL="https://fleet-manager.example.com" \
  --set token="registration-token" \
  --set clusterNamespace="fleet-default" \
  --set clusterName="my-cluster" \
  # Resource limits
  --set agent.resources.requests.cpu="100m" \
  --set agent.resources.requests.memory="128Mi" \
  --set agent.resources.limits.cpu="500m" \
  --set agent.resources.limits.memory="512Mi"
```

## Configuring Agent Resources

### Setting Resource Limits

```yaml
# fleet-agent-values.yaml
agent:
  # Resource allocation for the Fleet agent
  resources:
    requests:
      # Minimum CPU guaranteed to the agent
      cpu: "100m"
      # Minimum memory guaranteed
      memory: "128Mi"
    limits:
      # Maximum CPU the agent can use
      cpu: "500m"
      # Maximum memory the agent can use
      memory: "512Mi"
```

For edge clusters with limited resources:

```yaml
# fleet-agent-edge-values.yaml
agent:
  resources:
    requests:
      cpu: "50m"
      memory: "64Mi"
    limits:
      cpu: "200m"
      memory: "256Mi"
```

## Configuring Proxy Settings

For clusters behind an HTTP proxy:

```yaml
# fleet-agent-proxy-values.yaml
agent:
  extraEnv:
    # HTTP proxy for outbound traffic
    - name: HTTP_PROXY
      value: "http://proxy.corp.example.com:3128"
    # HTTPS proxy
    - name: HTTPS_PROXY
      value: "http://proxy.corp.example.com:3128"
    # Bypass proxy for local traffic
    - name: NO_PROXY
      value: "localhost,127.0.0.1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.cluster.local"
```

```bash
# Apply proxy settings during installation
helm install fleet-agent fleet/fleet-agent \
  --namespace cattle-fleet-system \
  --create-namespace \
  --set apiServerURL="https://fleet-manager.example.com" \
  --set token="registration-token" \
  -f fleet-agent-proxy-values.yaml
```

## Configuring Agent Tolerations

For nodes with taints (edge devices, GPU nodes, etc.):

```yaml
# fleet-agent-tolerations.yaml
agent:
  tolerations:
    # Allow running on nodes with the edge taint
    - key: "node.kubernetes.io/edge"
      operator: "Exists"
      effect: "NoSchedule"
    # Allow running on master/control-plane nodes
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"
```

## Configuring Agent Node Selector

Pin the Fleet agent to specific nodes:

```yaml
# fleet-agent-nodeselector.yaml
agent:
  nodeSelector:
    # Run the agent only on nodes labeled as system nodes
    node-role: system

  # Or use affinity for more complex rules
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
```

## Configuring TLS and Certificate Authorities

For environments with custom CA certificates:

```yaml
# fleet-agent-tls.yaml
# Add custom CA bundle for Fleet manager's TLS certificate
global:
  cattle:
    # CA bundle for the Rancher/Fleet manager certificate
    systemDefaultRegistry: ""

# Set the CA bundle
agent:
  extraEnv:
    - name: SSL_CERT_FILE
      value: /etc/ssl/certs/custom-ca.crt
  extraVolumeMounts:
    - name: custom-ca
      mountPath: /etc/ssl/certs/custom-ca.crt
      subPath: custom-ca.crt
  extraVolumes:
    - name: custom-ca
      configMap:
        name: custom-ca-bundle
```

## Updating Agent Configuration

For existing Fleet agents, update via Helm upgrade:

```bash
# Update resource limits on existing agent
helm upgrade fleet-agent fleet/fleet-agent \
  --namespace cattle-fleet-system \
  --reuse-values \
  --set agent.resources.limits.memory="1Gi"

# Update proxy settings
helm upgrade fleet-agent fleet/fleet-agent \
  --namespace cattle-fleet-system \
  --reuse-values \
  -f fleet-agent-proxy-values.yaml
```

## Configuring Agent Labels and Annotations

Propagate labels from the agent to the cluster registration:

```yaml
# fleet-agent-labels.yaml
# These become labels on the Fleet Cluster resource
clusterLabels:
  environment: production
  region: us-west-2
  team: platform
  managed-by: fleet-agent-helm

clusterAnnotations:
  installed-by: "platform-team"
  install-date: "2026-03-20"
```

## Verifying Agent Configuration

```bash
# Check the Fleet agent is running
kubectl get pods -n cattle-fleet-system

# View agent configuration
kubectl get deployment fleet-agent \
  -n cattle-fleet-system \
  -o yaml

# Check agent resource usage
kubectl top pods -n cattle-fleet-system

# View agent logs
kubectl logs -n cattle-fleet-system \
  -l app=fleet-agent \
  --tail=50

# Verify agent is connected to Fleet manager
kubectl get cluster -n fleet-default \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.status.agent.lastSeen}{"\n"}{end}'
```

## Troubleshooting Agent Issues

```bash
# Agent not starting - check pod events
kubectl describe pod -n cattle-fleet-system \
  -l app=fleet-agent

# Network connectivity issues
kubectl exec -n cattle-fleet-system \
  -l app=fleet-agent \
  -- curl -k https://fleet-manager.example.com/healthz

# Check agent version
kubectl get deployment fleet-agent \
  -n cattle-fleet-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

## Conclusion

Properly configuring the Fleet agent ensures reliable operation in diverse environments - from resource-constrained edge devices to enterprise clusters behind corporate proxies. By setting appropriate resource limits, configuring proxy settings where needed, and applying the right tolerations and node selectors, you ensure the Fleet agent runs stably and connects reliably to the Fleet manager. Regular verification of agent health and periodic updates keep your management infrastructure current and operational.
