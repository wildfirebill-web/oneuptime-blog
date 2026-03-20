# How to Configure Kubewarden Policy Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Kubernetes, PolicyServer, Security, Configuration

Description: Learn how to configure and manage Kubewarden PolicyServer resources to control how policies are loaded, executed, and scaled in your cluster.

## Introduction

The Kubewarden PolicyServer is the component that actually runs your WebAssembly policies and serves the admission webhook endpoint. You can configure multiple PolicyServers to isolate different sets of policies, scale them independently, and customize their resource allocation. Understanding how to configure PolicyServers is key to running Kubewarden efficiently in production.

## Prerequisites

- Kubewarden controller installed
- `kubectl` access with cluster-admin permissions
- Basic understanding of Kubernetes deployments

## Understanding PolicyServer Architecture

A PolicyServer is backed by a Kubernetes Deployment. When the Kubewarden controller sees an `AdmissionPolicy` or `ClusterAdmissionPolicy` that references a PolicyServer, it:
1. Loads the Wasm policy into the PolicyServer's memory
2. Registers a webhook that routes requests to this PolicyServer
3. The PolicyServer evaluates each admission request against the policy

## Default PolicyServer

Kubewarden ships with a default PolicyServer configured via the `kubewarden-defaults` Helm chart:

```bash
# View the default PolicyServer
kubectl get policyserver default -o yaml

# Check the default PolicyServer's status
kubectl describe policyserver default
```

## Creating a Custom PolicyServer

### Basic PolicyServer Configuration

```yaml
# policyserver-custom.yaml
apiVersion: policies.kubewarden.io/v1
kind: PolicyServer
metadata:
  name: strict-security
  namespace: kubewarden
spec:
  # Container image for the policy server
  image: ghcr.io/kubewarden/policy-server:latest

  # Number of replicas for HA
  replicas: 2

  # Service account for the PolicyServer pods
  serviceAccountName: kubewarden-policy-server-strict-security
```

```bash
# Apply the custom PolicyServer
kubectl apply -f policyserver-custom.yaml

# Check that it's running
kubectl get policyserver strict-security -n kubewarden
kubectl get pods -n kubewarden -l app=kubewarden-policy-server-strict-security
```

### High-Availability PolicyServer

```yaml
# policyserver-ha.yaml
apiVersion: policies.kubewarden.io/v1
kind: PolicyServer
metadata:
  name: production
  namespace: kubewarden
spec:
  image: ghcr.io/kubewarden/policy-server:v1.10.0

  # Run 3 replicas for HA
  replicas: 3

  # Resource allocation per replica
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"

  # Custom environment variables
  env:
    - name: KUBEWARDEN_LOG_LEVEL
      value: "info"
    - name: KUBEWARDEN_LOG_FMT
      value: "json"

  # Affinity for spreading replicas across nodes
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: kubewarden-policy-server-production
            topologyKey: kubernetes.io/hostname

  # Tolerations for dedicated policy server nodes
  tolerations:
    - key: "kubewarden.io/policy-server"
      operator: "Exists"
      effect: "NoSchedule"
```

## Assigning Policies to Specific PolicyServers

When creating a policy, specify which PolicyServer should run it:

```yaml
# policy-on-custom-server.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: strict-no-privileged
spec:
  module: registry://ghcr.io/kubewarden/policies/pod-privileged:v0.2.0

  # Route this policy to the strict-security PolicyServer
  policyServer: strict-security

  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
  mode: protect
```

## Configuring PolicyServer TLS

```yaml
# policyserver-tls.yaml
apiVersion: policies.kubewarden.io/v1
kind: PolicyServer
metadata:
  name: secure-server
  namespace: kubewarden
spec:
  image: ghcr.io/kubewarden/policy-server:v1.10.0
  replicas: 2

  # TLS configuration using cert-manager
  # Kubewarden manages TLS certificates automatically via cert-manager
  # You can also provide your own certificate
  volumes:
    - name: custom-ca
      configMap:
        name: custom-ca-bundle
  volumeMounts:
    - name: custom-ca
      mountPath: /etc/ssl/custom-ca
```

## Configuring Pull Secrets for Private OCI Registries

```yaml
# policyserver-private-registry.yaml
apiVersion: policies.kubewarden.io/v1
kind: PolicyServer
metadata:
  name: default
  namespace: kubewarden
spec:
  image: ghcr.io/kubewarden/policy-server:v1.10.0
  replicas: 2

  # Secret containing credentials for private policy registries
  imagePullSecret: private-registry-credentials
```

```bash
# Create the image pull secret
kubectl create secret docker-registry private-registry-credentials \
  --docker-server=registry.internal.example.com \
  --docker-username=kubewarden-bot \
  --docker-password=my-registry-token \
  -n kubewarden
```

## Monitoring PolicyServer Health

```bash
# Check PolicyServer status
kubectl get policyserver -n kubewarden

# Detailed status
kubectl describe policyserver default -n kubewarden

# Check the underlying pods
kubectl get pods -n kubewarden -l app=kubewarden-policy-server-default

# View PolicyServer logs
kubectl logs -n kubewarden \
  -l app=kubewarden-policy-server-default \
  --tail=50

# Check metrics (if Prometheus is configured)
kubectl port-forward -n kubewarden \
  svc/kubewarden-controller-metrics 8080:8080
curl http://localhost:8080/metrics | grep kubewarden
```

## Scaling PolicyServers

```bash
# Scale up the default PolicyServer for high traffic
kubectl patch policyserver default -n kubewarden \
  --type=merge \
  -p '{"spec":{"replicas":5}}'

# Scale back down after traffic decreases
kubectl patch policyserver default -n kubewarden \
  --type=merge \
  -p '{"spec":{"replicas":2}}'
```

## Upgrading PolicyServer

```bash
# Update the policy server image version
kubectl patch policyserver default -n kubewarden \
  --type=merge \
  -p '{"spec":{"image":"ghcr.io/kubewarden/policy-server:v1.11.0"}}'

# Watch the rollout
kubectl rollout status deployment \
  kubewarden-policy-server-default \
  -n kubewarden
```

## Conclusion

The Kubewarden PolicyServer is a highly configurable component that can be tuned for your specific production requirements. By creating dedicated PolicyServers for different policy categories, configuring appropriate resource limits, and enabling HA with multiple replicas, you ensure that your admission control infrastructure is as robust and scalable as your applications. Proper PolicyServer configuration is the foundation of a reliable Kubewarden deployment that can handle production admission traffic without becoming a bottleneck.
