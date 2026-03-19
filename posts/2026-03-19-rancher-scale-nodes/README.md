# How to Scale Cluster Nodes Up and Down in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Management, Node Management

Description: Learn how to scale your Kubernetes cluster nodes up and down using Rancher for both provisioned and cloud-managed clusters.

Scaling cluster nodes is a common operational task as workload demands change. Rancher provides several ways to scale nodes up and down, depending on how your cluster was provisioned. This guide covers scaling for Rancher-provisioned clusters, hosted cloud clusters, and custom clusters.

## Prerequisites

- Administrative access to the Rancher UI
- A running cluster managed by Rancher
- For cloud-provisioned clusters, valid cloud credentials with scaling permissions

## Scaling Rancher-Provisioned Clusters (Node Driver)

For clusters provisioned by Rancher using node drivers (vSphere, Harvester, cloud node drivers), scaling is managed through machine pools.

### Scale Up

1. Log in to the Rancher UI
2. Go to **Cluster Management**
3. Click on the cluster name
4. Navigate to **Machine Pools** (or **Nodes**)
5. Find the machine pool you want to scale
6. Click the **+** button or edit the machine pool
7. Increase the **Machine Count**
8. Click **Save**

Rancher will automatically provision new VMs and add them to the cluster.

### Scale Down

1. Navigate to the machine pool
2. Decrease the **Machine Count**
3. Click **Save**

Rancher will cordon, drain, and remove nodes automatically.

### Via the Cluster Configuration

You can also edit the cluster configuration directly:

1. Go to **Cluster Management**
2. Click the three-dot menu next to your cluster
3. Select **Edit Config**
4. Adjust the machine count in each pool
5. Click **Save**

## Scaling Hosted Cloud Clusters (EKS, GKE, AKS)

### Scaling EKS Node Groups

1. Navigate to the EKS cluster in Rancher
2. Click **Edit Config**
3. Find the **Node Groups** section
4. Adjust the **Desired Size**, **Minimum Size**, and **Maximum Size**
5. Click **Save**

Or from the command line:

```bash
aws eks update-nodegroup-config \
  --cluster-name <CLUSTER_NAME> \
  --nodegroup-name <NODEGROUP_NAME> \
  --scaling-config desiredSize=5,minSize=3,maxSize=10 \
  --region <REGION>
```

### Scaling GKE Node Pools

1. Navigate to the GKE cluster in Rancher
2. Click **Edit Config**
3. Find the **Node Pools** section
4. Adjust the node count or autoscaling settings
5. Click **Save**

Or via gcloud:

```bash
gcloud container clusters resize <CLUSTER_NAME> \
  --node-pool <POOL_NAME> \
  --num-nodes 5 \
  --region <REGION>
```

### Scaling AKS Node Pools

1. Navigate to the AKS cluster in Rancher
2. Click **Edit Config**
3. Adjust node pool count or autoscaling
4. Click **Save**

Or via Azure CLI:

```bash
az aks nodepool scale \
  --resource-group <RESOURCE_GROUP> \
  --cluster-name <CLUSTER_NAME> \
  --name <NODEPOOL_NAME> \
  --node-count 5
```

## Scaling Custom Clusters

For custom clusters where you manually registered nodes, scaling requires manual intervention.

### Scale Up

1. Go to the cluster in Rancher
2. Navigate to **Registration** or **Node** settings
3. Copy the registration command for the desired node role
4. Run the registration command on the new node

```bash
curl -fL https://rancher.yourdomain.com/system-agent-install.sh | sudo sh -s - \
  --server https://rancher.yourdomain.com \
  --label 'cattle.io/os=linux' \
  --token <TOKEN> \
  --worker
```

### Scale Down

1. Cordon the node to prevent new pod scheduling:

```bash
kubectl cordon <NODE_NAME>
```

2. Drain the node to evict workloads:

```bash
kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data
```

3. Delete the node from Kubernetes:

```bash
kubectl delete node <NODE_NAME>
```

4. In Rancher, the node will be removed from the cluster view.

5. Clean up the Rancher agent on the removed node:

```bash
# On the node being removed
sudo systemctl stop rancher-system-agent
sudo systemctl disable rancher-system-agent
sudo rm -rf /var/lib/rancher
```

## Configuring Cluster Autoscaler

For automated scaling based on workload demand, set up the Kubernetes Cluster Autoscaler.

### EKS Autoscaler

The EKS autoscaler adjusts node group sizes based on pending pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<CLUSTER_NAME>
```

### GKE and AKS

GKE and AKS have built-in autoscaler support. Enable it through the Rancher UI when editing the cluster configuration:

- Set **Enable Autoscaling** to true
- Configure **Min Nodes** and **Max Nodes**

## Monitoring Scaling Operations

### Check Node Status

After scaling, verify nodes are ready:

```bash
kubectl get nodes -o wide
kubectl describe nodes | grep -A5 "Conditions"
```

### Check Workload Distribution

Verify workloads are properly distributed after scaling:

```bash
kubectl get pods -A -o wide | grep -v Completed
```

### Monitor Resource Utilization

Check if scaling was effective:

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
```

## Best Practices

- Scale worker nodes, not control plane nodes, for workload changes. Control plane scaling should be planned separately.
- Use autoscaling for dynamic workloads instead of manual scaling.
- Always drain nodes before removing them to minimize workload disruption.
- Monitor pod disruption budgets to ensure scaling down does not violate availability requirements.
- Set resource requests and limits on pods so the autoscaler can make informed decisions.
- Use node pools or machine pools to separate different workload types and scale them independently.

## Troubleshooting

- **New nodes not joining**: Check the registration command, network connectivity, and node prerequisites.
- **Pods not scheduling on new nodes**: Check for taints, node selectors, or affinity rules that prevent scheduling.
- **Drain stuck**: Pods with restrictive PDB or local storage may block draining. Use `--force` as a last resort.
- **Autoscaler not scaling**: Check autoscaler logs and ensure pending pods have resource requests defined.

## Conclusion

Scaling nodes in Rancher adapts to how your cluster was provisioned. Machine pools make scaling Rancher-provisioned clusters as simple as changing a number. Cloud-managed clusters scale through their respective node group or pool configurations. Custom clusters require manual node addition and removal. For dynamic workloads, configure the cluster autoscaler to handle scaling automatically based on demand.
