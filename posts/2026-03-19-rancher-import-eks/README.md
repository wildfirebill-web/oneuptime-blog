# How to Import an EKS Cluster into Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, EKS, Cluster Management

Description: A practical guide to importing an existing Amazon EKS cluster into Rancher for unified multi-cluster management.

Amazon EKS is one of the most popular managed Kubernetes services. Importing your EKS clusters into Rancher gives you a unified management plane across all your clusters, enhanced RBAC, and integrated monitoring. This guide walks you through the process of importing an existing EKS cluster into Rancher.

## Prerequisites

- A running Rancher installation (v2.7 or later)
- An existing EKS cluster in AWS
- AWS CLI configured with appropriate permissions
- `kubectl` installed and configured for the EKS cluster
- Network connectivity from the EKS cluster to the Rancher server

## Step 1: Configure kubectl for the EKS Cluster

If you have not already configured kubectl for your EKS cluster, update your kubeconfig:

```bash
aws eks update-kubeconfig --name <EKS_CLUSTER_NAME> --region <REGION>
```

Verify access:

```bash
kubectl get nodes
kubectl cluster-info
```

## Step 2: Verify IAM Permissions

Your AWS IAM user or role needs the following permissions for the import:

- `eks:DescribeCluster`
- `eks:ListClusters`

The user running kubectl must be mapped in the EKS cluster's `aws-auth` ConfigMap with cluster-admin permissions.

Check your identity:

```bash
aws sts get-caller-identity
```

Verify you have cluster-admin access:

```bash
kubectl auth can-i '*' '*' --all-namespaces
```

## Step 3: Import the EKS Cluster in Rancher

### Option A: Import as an EKS Cluster (Recommended)

This option gives Rancher full visibility into the EKS-specific configuration.

1. Log in to the Rancher UI
2. Go to **Cluster Management**
3. Click **Import Existing**
4. Select **Amazon EKS**

### Configure AWS Credentials

Rancher needs AWS credentials to manage the EKS cluster. You can provide them in two ways:

**Option 1: Cloud Credential**

Create a cloud credential in Rancher:

1. Go to **Cluster Management > Cloud Credentials**
2. Click **Create**
3. Select **Amazon**
4. Enter your AWS Access Key and Secret Key
5. Save the credential

**Option 2: IAM Role (EC2 Instance Profile)**

If Rancher runs on EC2 instances, you can use an IAM instance profile attached to the Rancher nodes. Configure the IAM role with the necessary EKS permissions.

### Select the Cluster

After configuring credentials:

1. Select the AWS region where your EKS cluster is running
2. Rancher will list available EKS clusters
3. Select the cluster you want to import
4. Click **Register**

### Option B: Import as a Generic Cluster

If you prefer a simpler import without EKS-specific management:

1. Go to **Cluster Management**
2. Click **Import Existing**
3. Select **Generic**
4. Name the cluster
5. Copy the generated kubectl command

Apply the command on the EKS cluster:

```bash
kubectl apply -f https://rancher.yourdomain.com/v3/import/<token>.yaml
```

## Step 4: Wait for the Import to Complete

The import process deploys Rancher agents into the EKS cluster:

```bash
kubectl get pods -n cattle-system -w
kubectl get pods -n cattle-fleet-system -w
```

In the Rancher UI, watch the cluster status change from `Waiting` to `Provisioning` to `Active`.

## Step 5: Verify the Import

Once the cluster is Active:

```bash
# Check agent status
kubectl get deployments -n cattle-system

# Verify all nodes are visible in Rancher
kubectl get nodes
```

In the Rancher UI:

- Click on the imported EKS cluster
- Verify the cluster dashboard shows correct node count and Kubernetes version
- Check that EKS-specific details are visible (if imported as an EKS cluster)

## Step 6: Configure EKS-Specific Settings

When imported as an EKS cluster, Rancher provides additional management capabilities:

### View EKS Configuration

Navigate to the cluster and check the EKS configuration tab to see:

- Node groups and their configurations
- Kubernetes version
- VPC and subnet configuration
- Logging settings

### Manage Node Groups

From Rancher, you can view and manage EKS managed node groups:

1. Go to your EKS cluster in Rancher
2. Navigate to **Nodes**
3. View node group details and scaling configurations

### Upgrade Kubernetes Version

If the cluster was imported as an EKS type, you can trigger Kubernetes version upgrades from Rancher:

1. Go to the cluster configuration
2. Select a new Kubernetes version
3. Save the changes

## Step 7: Set Up Monitoring and Logging

Install the Rancher monitoring stack for unified observability:

1. Navigate to the EKS cluster in Rancher
2. Go to **Apps**
3. Install the **Monitoring** chart

For logging, you can also enable EKS control plane logging through the cluster configuration or install the Rancher logging operator.

## Step 8: Configure RBAC

Set up access control for the imported cluster:

1. Go to **Cluster Members**
2. Add users or groups with roles like Cluster Owner or Cluster Member
3. Create projects to organize namespaces

Rancher's RBAC works alongside EKS IAM-based authentication. Users can authenticate through Rancher or directly through AWS IAM.

## Networking Considerations

### VPC Configuration

Ensure the EKS cluster's VPC allows outbound connectivity to the Rancher server URL on port 443. If Rancher is in a different VPC, set up VPC peering or use a transit gateway.

### Security Groups

The EKS worker nodes need outbound HTTPS access to Rancher. No inbound rules are required as the agent initiates the connection.

### Private EKS Clusters

For private EKS clusters with no public endpoint:

1. Ensure Rancher can reach the EKS API endpoint through VPC peering or a VPN
2. The Rancher agents inside the EKS cluster must be able to reach the Rancher server URL
3. You may need to configure proxy settings on the agents

## Troubleshooting

- **Import fails with permissions error**: Verify your AWS credentials and IAM permissions. Check that the user is mapped in the `aws-auth` ConfigMap.
- **Agent cannot connect to Rancher**: Check VPC security groups and network ACLs. Verify outbound HTTPS is allowed.
- **Cluster shows as Unavailable**: Check agent logs with `kubectl logs -l app=cattle-cluster-agent -n cattle-system`.
- **EKS details not visible**: Make sure you imported the cluster as an EKS type, not Generic, and that the cloud credential has the correct permissions.

## Conclusion

Importing EKS clusters into Rancher gives you centralized management across your AWS Kubernetes infrastructure and any other clusters you manage. The EKS-specific import option provides deeper integration with node group management and version upgrades, while the generic import works for basic management needs. Once imported, you benefit from Rancher's unified RBAC, monitoring, and multi-cluster management capabilities.
