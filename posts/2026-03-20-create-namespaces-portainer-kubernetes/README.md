# How to Create Namespaces in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Namespaces, Multi-Tenancy, DevOps

Description: Learn how to create and configure Kubernetes namespaces in Portainer for team and workload isolation.

## What Are Namespaces?

Namespaces provide a mechanism for isolating groups of resources within a cluster. They're used to separate workloads by team, environment (dev/staging/prod), or application component. Each namespace has its own:

- Resources (pods, services, deployments)
- Resource quotas
- Access control (RBAC)
- Network policies

## Creating a Namespace in Portainer

1. Select your Kubernetes environment.
2. Go to **Namespaces** in the sidebar.
3. Click **Add namespace**.
4. Enter the namespace **name** (lowercase, no spaces or underscores).
5. Optionally configure:
   - **Resource quotas**: CPU and memory limits for the namespace.
   - **Load balancer access**: Allow users to create LoadBalancer services.
   - **Ingress access**: Allow or restrict Ingress creation.
6. Click **Create namespace**.

## Namespace Naming Conventions

```bash
# Good namespace names
production
staging
development
team-frontend
team-backend
monitoring
logging

# Invalid names (no uppercase, no underscores)
Production          # Invalid: uppercase
my_namespace        # Invalid: underscore
my namespace        # Invalid: space
```

## Creating a Namespace via CLI

```bash
# Create a namespace
kubectl create namespace production

# Create with a label for better organization
kubectl create namespace production --dry-run=client -o yaml | \
  kubectl label --local -f - \
  environment=production \
  team=platform -o yaml | \
  kubectl apply -f -

# Or use a manifest file
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
EOF
```

## Setting Resource Quotas During Creation

In Portainer, enable resource quotas when creating the namespace:

```yaml
# Equivalent ResourceQuota created by Portainer
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "100"
    services: "30"
```

## Viewing Namespace Details

```bash
# List all namespaces
kubectl get namespaces

# Describe a namespace including quota usage
kubectl describe namespace production

# Show current quota usage
kubectl get resourcequota --namespace=production
```

## Namespace Best Practices

- Use separate namespaces for each environment (dev, staging, production).
- Apply resource quotas to prevent a single team from consuming all cluster resources.
- Use RBAC to limit what users can do per namespace.
- Label namespaces consistently for policy enforcement.

## Conclusion

Namespaces are the foundation of multi-tenancy in Kubernetes. Portainer makes namespace creation and management straightforward, with built-in support for resource quotas and access control settings at creation time.
