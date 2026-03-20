# How to Set Up Cluster Registry Access in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Container Registry, Image Pull Secrets, DevOps

Description: Learn how to configure cluster-wide registry access in Portainer so all namespaces can pull from private registries without manual secret creation.

## The Challenge

In Kubernetes, `imagePullSecrets` are namespace-scoped. Without tooling, you must create the same pull secret in every namespace that needs it. Portainer solves this by managing the secret distribution automatically.

## How Portainer Manages Registry Secrets

When you assign a registry to a Kubernetes environment in Portainer, it:
1. Creates a `Secret` of type `kubernetes.io/dockerconfigjson` in the target namespace.
2. Patches the namespace's `default` service account to reference the secret.
3. All pods in that namespace can then pull from the registry without explicit configuration.

## Configuring Cluster Registry Access

### Step 1: Add the Registry

Go to **Settings > Registries** and add your private registry with credentials.

### Step 2: Assign the Registry to the Environment

1. Go to **Environments**, select your Kubernetes cluster.
2. Click **Edit**.
3. Enable the registry under **Registries**.
4. Choose which namespaces the registry is available to.

### Step 3: (Optional) Grant Access Per Namespace

For namespace-level control:

1. Go to **Namespaces**, select a namespace.
2. Under **Registries**, enable or disable specific registries for that namespace.

## Manual imagePullSecret Management (CLI Reference)

```bash
# Create an imagePullSecret in a namespace

kubectl create secret docker-registry registry-credentials \
  --docker-server=registry.mycompany.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --namespace=my-namespace

# Patch the default service account to use the secret
kubectl patch serviceaccount default \
  -n my-namespace \
  -p '{"imagePullSecrets": [{"name": "registry-credentials"}]}'

# Copy a secret to another namespace
kubectl get secret registry-credentials \
  -n source-namespace \
  -o yaml | \
  sed 's/namespace: source-namespace/namespace: target-namespace/' | \
  kubectl apply -f -
```

## Automating Secret Propagation

For clusters not using Portainer, tools like `kubernetes-reflector` can mirror secrets across namespaces:

```yaml
# Annotate a secret to be reflected to all namespaces
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: portainer
  annotations:
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
```

## Verifying Registry Access

```bash
# Check that the imagePullSecret exists in a namespace
kubectl get secrets -n my-namespace | grep registry

# Confirm the default service account has the pull secret attached
kubectl get serviceaccount default -n my-namespace -o yaml | grep imagePullSecrets -A5

# Check pod events if image pull is failing
kubectl describe pod <pod-name> -n my-namespace | grep -A5 Events
```

## Conclusion

Portainer eliminates the manual work of creating and distributing registry secrets across namespaces. By managing `imagePullSecrets` centrally, it ensures consistent access control while reducing operational overhead.
