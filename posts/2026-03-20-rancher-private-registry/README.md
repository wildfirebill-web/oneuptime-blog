# How to Configure a Private Registry in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Container Registry, Private Registry

Description: Learn how to configure and use a private container registry in Rancher to securely pull and manage container images across your clusters.

## Introduction

Private container registries are essential for organizations that need to maintain control over their container images, enforce security policies, or operate in air-gapped environments. Rancher provides built-in support for connecting to private registries, making it straightforward to pull images from internal or external private repositories.

This guide walks you through configuring a private registry in Rancher, setting up authentication, and ensuring your workloads can pull images seamlessly.

## Prerequisites

- A running Rancher instance (v2.6 or later)
- Access to a private container registry (self-hosted or cloud-based)
- `kubectl` configured to talk to your cluster
- Rancher cluster admin or project owner permissions

## Understanding Registry Authentication in Kubernetes

Kubernetes uses `imagePullSecrets` to authenticate with private registries. These secrets contain the registry URL, username, and password encoded in a Docker configuration format. Rancher provides a UI-driven approach to manage these secrets at both the cluster and project level.

## Step 1: Add a Private Registry to a Rancher Project

1. Log in to the Rancher UI.
2. Navigate to your cluster and select a **Project**.
3. Go to **Secrets** > **Registry Credentials**.
4. Click **Add Registry**.
5. Fill in the form:
   - **Name**: A descriptive name for the registry
   - **Scope**: Project or Namespace
   - **Address**: Your registry URL (e.g., `registry.example.com`)
   - **Username**: Registry username
   - **Password**: Registry password or token

Rancher automatically creates a Kubernetes `Secret` of type `kubernetes.io/dockerconfigjson` in the target namespace.

## Step 2: Create a Registry Secret via kubectl

You can also create registry secrets directly with kubectl:

```bash
# Create a registry secret for a private registry
kubectl create secret docker-registry my-private-registry \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=admin@example.com \
  --namespace=my-namespace
```

Verify the secret was created:

```bash
kubectl get secret my-private-registry -n my-namespace -o yaml
```

## Step 3: Configure a Deployment to Use the Registry

Reference the registry secret in your deployment manifest:

```yaml
# deployment.yaml - Deployment using a private registry image
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      # Reference the registry secret here
      imagePullSecrets:
        - name: my-private-registry
      containers:
        - name: my-app
          # Use the full registry URL for the image
          image: registry.example.com/myorg/my-app:v1.0.0
          ports:
            - containerPort: 8080
```

## Step 4: Set Default Registry in Rancher

For clusters where all images come from a single private registry, you can configure a default registry:

1. In Rancher UI, navigate to **Cluster Management** > your cluster > **Edit Config**.
2. Under **Advanced Options**, find **Private Registry**.
3. Enter your registry URL and credentials.

Alternatively, configure this in RKE2 cluster configuration:

```yaml
# rancher-cluster.yaml - RKE2 cluster with private registry
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: my-cluster
  namespace: fleet-default
spec:
  rkeConfig:
    registries:
      configs:
        # Configure authentication for the private registry
        registry.example.com:
          authConfigSecretName: registry-secret
          insecureSkipVerify: false
      mirrors:
        # Mirror docker.io images through your private registry
        docker.io:
          endpoints:
            - https://registry.example.com
```

## Step 5: Automate Secret Distribution with ServiceAccount

To automatically inject registry secrets into all pods in a namespace, patch the default ServiceAccount:

```bash
# Patch the default service account to automatically use the registry secret
kubectl patch serviceaccount default \
  -n my-namespace \
  -p '{"imagePullSecrets": [{"name": "my-private-registry"}]}'
```

Any pod using the default ServiceAccount in that namespace will automatically have access to the private registry.

## Troubleshooting Common Issues

### ImagePullBackOff Error

If you see `ImagePullBackOff` or `ErrImagePull`:

```bash
# Check pod events for detailed error messages
kubectl describe pod <pod-name> -n <namespace>

# Verify the secret exists and has correct credentials
kubectl get secret my-private-registry -n my-namespace -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

### Testing Registry Connectivity

```bash
# Test registry authentication from inside the cluster
kubectl run test-registry --image=registry.example.com/myorg/test-image:latest \
  --restart=Never \
  --overrides='{"spec":{"imagePullSecrets":[{"name":"my-private-registry"}]}}'
```

## Conclusion

Configuring a private registry in Rancher is straightforward and provides secure, centralized management of container images. By leveraging Rancher's built-in registry credential management, you can ensure your workloads consistently pull from the correct sources while maintaining authentication security. For production environments, consider using a dedicated secrets management solution like Vault to rotate registry credentials automatically.
