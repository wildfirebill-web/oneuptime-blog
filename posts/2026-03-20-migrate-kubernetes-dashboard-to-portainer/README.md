# How to Migrate from Kubernetes Dashboard to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Portainer, Migration, Container Management, DevOps, Dashboard

Description: Learn how to migrate your Kubernetes management workflow from the native Kubernetes Dashboard to Portainer for a richer feature set and unified multi-environment control.

---

Kubernetes Dashboard is minimal by design — it gives you visibility but not much control. Portainer goes further: stack deployments, registry management, role-based access, and a consistent UI across Docker and Kubernetes environments. This guide walks you through migrating from one to the other.

---

## Why Move to Portainer?

| Feature | Kubernetes Dashboard | Portainer |
|---|---|---|
| Stack/Compose support | No | Yes |
| Registry management | No | Yes |
| RBAC | Basic | Advanced |
| Multi-environment | No | Yes |
| Helm charts | No | Yes (BE) |

---

## Step 1: Remove the Kubernetes Dashboard (Optional)

If you no longer want the old dashboard running, remove it cleanly before proceeding.

```bash
# Delete the Kubernetes Dashboard deployment and associated resources
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Confirm all dashboard resources are removed
kubectl get all -n kubernetes-dashboard
```

---

## Step 2: Deploy Portainer Agent on Your Cluster

Portainer uses an agent to communicate with your Kubernetes cluster. Deploy it using the official manifest.

```bash
# Apply the Portainer Agent manifest for Kubernetes
kubectl apply -n portainer -f https://downloads.portainer.io/ce2-21/portainer-agent-k8s-lb.yaml

# Verify the agent pod is running
kubectl get pods -n portainer
```

---

## Step 3: Install Portainer Server (If Not Already Running)

If you don't have a Portainer server running elsewhere, deploy it into the same cluster.

```bash
# Create the portainer namespace
kubectl create namespace portainer

# Deploy Portainer CE using the NodePort manifest
kubectl apply -n portainer -f https://downloads.portainer.io/ce2-21/portainer.yaml

# Get the service and find the NodePort
kubectl get svc -n portainer
```

---

## Step 4: Connect Your Cluster in Portainer

Once Portainer is running, add your Kubernetes environment via the UI or via the Portainer API.

```bash
# Get the agent service endpoint (use this URL in Portainer's "Add Environment" wizard)
kubectl get svc portainer-agent -n portainer -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

In the Portainer UI:
1. Navigate to **Settings > Environments > Add Environment**
2. Choose **Kubernetes via Agent**
3. Paste the agent endpoint URL
4. Click **Connect**

---

## Step 5: Recreate Your Workloads as Stacks

Portainer's stack feature lets you manage workloads using familiar YAML manifests — a big step up from the Kubernetes Dashboard.

```yaml
# example-stack.yaml — deploy an Nginx workload as a Portainer stack
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

In Portainer, go to **Stacks > Add Stack**, paste this YAML, and deploy.

---

## Step 6: Set Up Access Control

Portainer's RBAC model mirrors what Kubernetes offers but through a friendlier interface. Create teams and assign environment-level permissions.

1. Go to **Settings > Users** to create user accounts
2. Go to **Settings > Teams** to group users
3. In **Environments**, assign teams with specific roles (Operator, Helpdesk, etc.)

---

## Monitoring with OneUptime

Once Portainer is managing your workloads, integrate OneUptime to monitor service health across all your Kubernetes namespaces — giving you uptime checks, on-call alerts, and status pages in one place.

---

## Summary

Migrating from Kubernetes Dashboard to Portainer takes about 15 minutes and gives you a dramatically richer management experience. You gain stack deployments, registry management, and multi-environment support without losing any visibility the old dashboard provided.
