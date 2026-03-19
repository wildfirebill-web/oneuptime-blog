# How to Import an Existing Kubernetes Cluster into Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Management

Description: Learn how to import any existing Kubernetes cluster into Rancher for centralized management and monitoring.

If you already have Kubernetes clusters running outside of Rancher, you can import them into Rancher for centralized management without disrupting existing workloads. This guide shows you how to import any Kubernetes cluster, whether it is a managed cloud service, a self-hosted cluster, or a distribution like K3s or Kubeadm.

## Prerequisites

- A running Rancher installation (v2.7 or later)
- An existing Kubernetes cluster you want to import
- `kubectl` access to the target cluster with cluster-admin permissions
- Network connectivity from the target cluster to the Rancher server URL on port 443

## Step 1: Prepare the Target Cluster

Before importing, verify your target cluster is healthy:

```bash
# Switch to the target cluster's kubeconfig
export KUBECONFIG=/path/to/target-cluster-kubeconfig

# Verify cluster is accessible
kubectl cluster-info

# Check node status
kubectl get nodes

# Verify you have cluster-admin access
kubectl auth can-i '*' '*' --all-namespaces
```

## Step 2: Start the Import Process in Rancher

1. Log in to the Rancher UI
2. Click **Cluster Management** in the left sidebar
3. Click the **Import Existing** button
4. Select **Generic** for a standard Kubernetes cluster (or select the specific provider like EKS, GKE, AKS if applicable)

## Step 3: Configure Import Settings

### Cluster Name

Enter a descriptive name for the cluster as it will appear in Rancher (e.g., `prod-east-1` or `staging-cluster`).

### Labels and Annotations

Add labels to help organize and filter clusters:

```plaintext
environment: production
team: platform
region: us-east-1
```

## Step 4: Get the Import Command

Rancher generates a kubectl command to apply on your target cluster. The command deploys the Rancher agent, which establishes a connection back to the Rancher server.

You will see two options:

1. **Standard command** (for clusters with public CA certificates):

```bash
kubectl apply -f https://rancher.yourdomain.com/v3/import/<unique-token>.yaml
```

2. **Insecure command** (for self-signed certificates):

```bash
curl --insecure -sfL https://rancher.yourdomain.com/v3/import/<unique-token>.yaml | kubectl apply -f -
```

Use the insecure option only if your Rancher server uses self-signed certificates.

## Step 5: Apply the Import Command

Switch to the target cluster context and run the import command:

```bash
export KUBECONFIG=/path/to/target-cluster-kubeconfig

kubectl apply -f https://rancher.yourdomain.com/v3/import/<unique-token>.yaml
```

This creates several resources on the target cluster:

- A `cattle-system` namespace
- A `cattle-cluster-agent` deployment
- Required RBAC roles and bindings
- A `cattle-fleet-system` namespace (for Fleet integration)

## Step 6: Monitor the Import

Watch the agent pods deploy:

```bash
kubectl get pods -n cattle-system -w
```

You should see:

- `cattle-cluster-agent` pods starting up
- `cattle-fleet-agent` pods starting up

In the Rancher UI, the cluster status will progress through:

1. **Waiting**: Rancher is waiting for the agent to connect
2. **Provisioning**: The agent is deploying components
3. **Active**: The cluster is fully imported and managed

The import process typically takes 2 to 5 minutes.

## Step 7: Verify the Import

Once the cluster shows as `Active` in Rancher:

### From the Rancher UI

- Click on the imported cluster
- Verify the cluster dashboard shows node count, resource usage, and Kubernetes version
- Open the kubectl shell and run a test command
- Navigate to **Workloads** and verify existing deployments are visible

### From the Command Line

```bash
# Verify agents are running
kubectl get deployments -n cattle-system

# Check agent logs
kubectl logs -l app=cattle-cluster-agent -n cattle-system --tail=20

# Verify Fleet agent
kubectl get deployments -n cattle-fleet-system
```

## Step 8: Configure Post-Import Settings

### Set Up Projects and Namespaces

Imported clusters have a default project. Organize your existing namespaces into projects:

1. Navigate to the cluster in Rancher
2. Go to **Projects/Namespaces**
3. Create projects and move namespaces into them

### Configure RBAC

Set up role bindings to control who can access the imported cluster:

1. Go to **Cluster Members**
2. Add users or groups with appropriate roles (Cluster Owner, Cluster Member, or custom roles)

### Enable Monitoring

Install the Rancher monitoring stack for visibility:

1. Go to **Apps** in the cluster
2. Install the **Monitoring** chart
3. Configure Prometheus retention and Grafana dashboards

## Handling Special Cases

### Clusters Behind a Proxy

If the target cluster accesses the internet through a proxy, configure the agent to use it:

```bash
kubectl set env deployment/cattle-cluster-agent \
  HTTP_PROXY=http://proxy.company.com:3128 \
  HTTPS_PROXY=http://proxy.company.com:3128 \
  NO_PROXY=localhost,127.0.0.1,10.0.0.0/8 \
  -n cattle-system
```

### Clusters with Private Network

If the target cluster cannot reach Rancher directly, set up a network path (VPN, peering, or tunnel) between the cluster and the Rancher server.

### Re-importing a Cluster

If you need to re-import a cluster that was previously removed from Rancher, clean up the old agent resources first:

```bash
kubectl delete namespace cattle-system
kubectl delete namespace cattle-fleet-system
kubectl delete clusterrolebinding cattle-admin-binding
kubectl delete clusterrole cattle-admin
```

Then apply the new import command.

## Troubleshooting

- **Cluster stuck in Waiting**: The agent cannot reach Rancher. Check network connectivity with `curl -k https://rancher.yourdomain.com/healthz` from the target cluster.
- **Agent CrashLoopBackOff**: Check agent logs with `kubectl logs -l app=cattle-cluster-agent -n cattle-system`. Common causes include DNS resolution failures or TLS certificate issues.
- **Cluster shows as Unavailable**: The agent lost connectivity. Check the agent pod status and network path to Rancher.

## Conclusion

Importing existing Kubernetes clusters into Rancher provides centralized management, monitoring, and RBAC without disrupting your running workloads. The process is straightforward: start the import in the Rancher UI, apply the generated command on the target cluster, and verify the connection. Once imported, you get full visibility and management capabilities across all your clusters from a single interface.
