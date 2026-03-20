# How to Configure Per-Cluster Registry Access in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Container Registry, Access Control, DevOps

Description: Learn how to configure registry access on a per-Kubernetes-cluster basis in Portainer so different clusters use different registries.

## Overview

In multi-cluster Portainer setups, you may want different Kubernetes clusters to access different registries. For example, a production cluster might only access a private ECR registry, while a staging cluster can also pull from Docker Hub. Portainer lets you configure registry access per environment.

## Assigning Registries to a Kubernetes Environment

1. In Portainer, go to **Settings > Registries** and ensure all required registries are added.
2. Navigate to **Environments** and select a Kubernetes cluster.
3. Click **Edit**.
4. Scroll to the **Registries** section.
5. Enable only the registries this cluster should have access to.
6. Click **Update environment**.

## How Portainer Handles Registry Credentials in Kubernetes

When you deploy an application from Portainer to Kubernetes, it automatically creates a Kubernetes `imagePullSecret` using the stored registry credentials. This secret is injected into the deployment's pod spec.

```yaml
# Portainer creates a secret like this automatically
apiVersion: v1
kind: Secret
metadata:
  name: portainer-registry-secret
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  # Base64-encoded Docker config JSON with registry credentials
  .dockerconfigjson: <base64-encoded-credentials>
```

## Manually Creating an imagePullSecret (CLI Reference)

```bash
# Create an imagePullSecret for a private registry
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.mycompany.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@mycompany.com \
  --namespace=default

# Patch the default service account to use the secret automatically
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "my-registry-secret"}]}' \
  --namespace=default
```

## Per-Namespace Registry Access

For more granular control, configure registry access per namespace in Portainer:

1. Go to your Kubernetes environment.
2. Navigate to **Namespaces** and select a namespace.
3. Under **Registries**, enable specific registries for that namespace only.

## Verifying Registry Access

```bash
# Check if the imagePullSecret was created
kubectl get secrets --all-namespaces | grep registry

# Describe a pod to confirm imagePullSecrets are attached
kubectl describe pod <pod-name> | grep "Image Pull Secrets"
```

## Conclusion

Per-cluster registry configuration in Portainer gives you environment-level control over which image sources are accessible. This is a key component of a defense-in-depth container security strategy.
