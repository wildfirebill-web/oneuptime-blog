# How to Delete a Cluster in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Management

Description: A guide to safely deleting clusters from Rancher, covering imported clusters, provisioned clusters, and proper cleanup procedures.

Deleting a cluster in Rancher is a straightforward operation, but understanding what happens behind the scenes is important to avoid unintended consequences. This guide covers how to properly delete different types of clusters from Rancher and what cleanup steps to take.

## Understanding Cluster Deletion Behavior

The behavior of cluster deletion depends on how the cluster was created:

- **Rancher-provisioned clusters** (custom, node driver): Rancher deletes the cluster and can clean up infrastructure resources (VMs, cloud resources)
- **Imported clusters**: Rancher removes its agents from the cluster but does not delete the cluster itself
- **Hosted clusters** (EKS, GKE, AKS provisioned by Rancher): Rancher can delete the cloud-managed cluster and its resources

## Prerequisites

- Administrative access to the Rancher UI
- `kubectl` access to both the Rancher management cluster and the target cluster (for manual cleanup)
- Understanding of which workloads are running on the cluster

## Step 1: Pre-Deletion Checklist

Before deleting any cluster, complete this checklist:

```plaintext
Pre-Deletion Checklist
======================
[ ] Confirm no critical workloads are running on the cluster
[ ] Back up any important data and configurations
[ ] Notify teams that use the cluster
[ ] Document the cluster configuration for potential recreation
[ ] Export any persistent data from the cluster
[ ] Verify you are deleting the correct cluster
```

### Back Up Important Data

Export cluster resources you might need later:

```bash
# Export all custom resources

kubectl get all -A -o yaml > cluster-resources-backup.yaml

# Export configmaps and secrets
kubectl get configmaps -A -o yaml > configmaps-backup.yaml
kubectl get secrets -A -o yaml > secrets-backup.yaml

# Export PVCs
kubectl get pvc -A -o yaml > pvcs-backup.yaml
```

## Step 2: Delete the Cluster from the Rancher UI

### For Any Cluster Type

1. Log in to the Rancher UI
2. Go to **Cluster Management**
3. Find the cluster you want to delete
4. Click the three-dot menu on the right side of the cluster row
5. Select **Delete**
6. Confirm the deletion when prompted

Rancher will ask you to type the cluster name to confirm the deletion for safety.

### What Happens Next

For **imported clusters**:

- Rancher removes the cluster from its management
- Rancher agents are removed from the cluster
- The Kubernetes cluster itself continues to run independently
- Workloads on the cluster are not affected

For **Rancher-provisioned clusters** (custom with node driver):

- Rancher removes the cluster from management
- If provisioned via a node driver (vSphere, Harvester), the VMs may be deleted
- Kubernetes components are torn down

For **hosted provider clusters** (EKS, GKE, AKS provisioned by Rancher):

- Rancher sends a delete request to the cloud provider
- The cloud provider tears down the cluster and associated resources
- Node groups, VMs, and load balancers are deleted
- Persistent volumes may or may not be deleted depending on the reclaim policy

## Step 3: Verify Deletion

### Check Rancher

Confirm the cluster no longer appears in **Cluster Management**.

Check for any lingering resources in the management cluster:

```bash
kubectl get clusters.management.cattle.io
kubectl get clusters.provisioning.cattle.io -A
```

### Check Cloud Resources (for provisioned clusters)

For EKS:

```bash
aws eks list-clusters --region <REGION>
```

For GKE:

```bash
gcloud container clusters list --project <PROJECT_ID>
```

For AKS:

```bash
az aks list --resource-group <RESOURCE_GROUP>
```

## Step 4: Clean Up Imported Clusters (If Needed)

When you delete an imported cluster from Rancher, the agents should be removed automatically. If they are not, clean them up manually:

```bash
# Switch to the formerly imported cluster's kubeconfig
kubectl delete namespace cattle-system
kubectl delete namespace cattle-fleet-system
kubectl delete namespace cattle-fleet-local-system

# Remove RBAC resources
kubectl delete clusterrolebinding cattle-admin-binding
kubectl delete clusterrole cattle-admin

# Remove any remaining Rancher CRDs
kubectl get crd | grep cattle.io | awk '{print $1}' | xargs kubectl delete crd
kubectl get crd | grep fleet.cattle.io | awk '{print $1}' | xargs kubectl delete crd
```

## Step 5: Clean Up Management Cluster Resources

After deleting a cluster, some resources may remain in the Rancher management cluster:

```bash
# Check for orphaned cluster resources
kubectl get clusters.management.cattle.io

# Check for orphaned namespaces (each cluster gets a namespace)
kubectl get namespaces | grep "^c-"

# Clean up if needed
kubectl delete namespace <orphaned-namespace>
```

## Step 6: Clean Up DNS and Load Balancers

If the deleted cluster had:

- DNS records pointing to cluster ingresses, remove them
- External load balancers, verify they were deleted
- Firewall rules specific to the cluster, clean them up

## Deleting Multiple Clusters

If you need to delete multiple clusters, you can do it through the Rancher UI by selecting multiple clusters:

1. Go to **Cluster Management**
2. Select the checkboxes next to each cluster you want to delete
3. Click the **Delete** button in the toolbar
4. Confirm the deletion

## Force Deleting a Stuck Cluster

Sometimes a cluster gets stuck in a `Removing` state. To force the deletion:

```bash
# Find the cluster resource
kubectl get clusters.management.cattle.io

# Remove the finalizer to allow deletion
kubectl patch clusters.management.cattle.io <CLUSTER_NAME> \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge
```

Note: Force deletion skips cleanup steps, so you may need to manually clean up infrastructure resources.

## Recovering from Accidental Deletion

If you accidentally deleted a cluster:

- **Imported cluster**: The cluster itself still exists. Re-import it into Rancher by following the import process again.
- **Provisioned cluster**: If the infrastructure was not yet deleted, you may be able to re-import it. Otherwise, you need to recreate the cluster and restore from backups.
- **Hosted cluster**: Check if the cloud provider has a deletion protection or recovery option. Some providers retain resources briefly after deletion.

## Best Practices

- Enable deletion protection on critical cloud-managed clusters (EKS, GKE, AKS) at the cloud provider level
- Always back up cluster data before deletion
- Use Rancher's RBAC to restrict who can delete clusters
- Tag clusters with metadata indicating their purpose and criticality
- Maintain an inventory of clusters and their configurations
- Test cluster recreation procedures regularly

## Conclusion

Deleting a cluster in Rancher is simple through the UI, but understanding the implications for different cluster types is essential. Imported clusters continue to exist after removal from Rancher, while provisioned and hosted clusters may be fully torn down. Always back up data, verify cleanup of infrastructure resources, and remove any orphaned DNS records or load balancers after deletion.
