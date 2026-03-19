# How to Provision an EKS Cluster from Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, EKS, Cluster Management

Description: Step-by-step guide to provisioning and managing an Amazon EKS cluster directly from the Rancher UI.

Rancher can provision fully managed EKS clusters directly in your AWS account, giving you the benefits of AWS's managed Kubernetes service with Rancher's unified management interface. This guide walks you through provisioning an EKS cluster from Rancher, including IAM setup, network configuration, and node group creation.

## Prerequisites

- A running Rancher installation (v2.7 or later)
- An AWS account with appropriate permissions
- AWS Access Key and Secret Key (or IAM role if Rancher runs on EC2)
- A VPC with subnets configured for EKS (or let Rancher create one)

## Step 1: Set Up AWS IAM Permissions

Create an IAM user or role with the permissions Rancher needs to provision EKS:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:*",
        "ec2:*",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:GetRole",
        "iam:PassRole",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:GetInstanceProfile",
        "iam:CreateServiceLinkedRole",
        "iam:ListAttachedRolePolicies",
        "cloudformation:*",
        "autoscaling:*",
        "elasticloadbalancing:*",
        "kms:DescribeKey",
        "logs:CreateLogGroup",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    }
  ]
}
```

## Step 2: Create a Cloud Credential in Rancher

1. Log in to the Rancher UI
2. Go to **Cluster Management > Cloud Credentials**
3. Click **Create**
4. Select **Amazon**
5. Enter your AWS Access Key ID and Secret Access Key
6. Optionally set a default region
7. Name the credential and click **Create**

## Step 3: Start the EKS Provisioning Wizard

1. Go to **Cluster Management**
2. Click **Create**
3. Select **Amazon EKS** under hosted Kubernetes providers

## Step 4: Configure the EKS Cluster

### Basic Settings

- **Cluster Name**: Enter a name (e.g., `production-eks`)
- **Cloud Credential**: Select the credential you created
- **Region**: Choose the AWS region

### Kubernetes Version

Select the EKS Kubernetes version. Choose a version supported by both AWS and your Rancher version.

### Networking

Configure the VPC and networking:

**Use Existing VPC:**

- Select your VPC from the dropdown
- Select at least 2 subnets in different availability zones
- Choose public subnets, private subnets, or both

**Create New VPC:**

- Let Rancher create a new VPC with the required subnets
- Specify the CIDR block (e.g., `10.0.0.0/16`)

### Public Access

Configure API server endpoint access:

- **Public**: API server accessible from the internet
- **Private**: API server accessible only within the VPC
- **Public and Private**: Both access methods enabled

```plaintext
Public Access Sources: 0.0.0.0/0 (or restrict to specific CIDRs)
```

### Logging

Enable EKS control plane logging:

- API server logs
- Audit logs
- Authenticator logs
- Controller manager logs
- Scheduler logs

### Secrets Encryption

Optionally enable envelope encryption for Kubernetes secrets using an AWS KMS key:

```plaintext
KMS Key ARN: arn:aws:kms:us-east-1:123456789:key/your-key-id
```

## Step 5: Configure Node Groups

### Managed Node Group Settings

Click **Add Node Group** and configure:

- **Name**: Node group name (e.g., `general-purpose`)
- **Instance Type**: Select the EC2 instance type (e.g., `m5.xlarge`)
- **Desired Size**: Number of nodes (e.g., 3)
- **Minimum Size**: Minimum for autoscaling (e.g., 2)
- **Maximum Size**: Maximum for autoscaling (e.g., 10)

### Disk Configuration

- **Disk Size**: Root volume size in GB (e.g., 50)
- **Disk Type**: gp3 (recommended) or gp2

### AMI Type

Select the AMI type:

- **Amazon Linux 2**: Default, most compatible
- **Bottlerocket**: Security-focused, minimal OS
- **Ubuntu**: Ubuntu-based nodes

### Spot Instances

To reduce costs, you can configure spot instances:

- **Request Spot Instances**: Enable to use spot pricing
- **Spot Instance Types**: Add fallback instance types

### Labels and Taints

Add labels to the node group:

```plaintext
environment: production
team: platform
```

Add taints if needed:

```plaintext
dedicated: gpu:NoSchedule
```

### SSH Access

Configure SSH key pair for node access:

- Select an existing EC2 key pair
- Specify security groups for SSH access

## Step 6: Create the Cluster

Review all settings and click **Create**. Rancher will:

1. Create the EKS cluster in AWS
2. Create the configured node groups
3. Deploy Rancher agents into the cluster
4. Register the cluster in Rancher

This process takes 10 to 20 minutes.

## Step 7: Monitor Provisioning

Watch the progress in the Rancher UI:

1. Navigate to your cluster
2. Monitor the provisioning status
3. Watch for any error messages

You can also check the AWS CloudFormation console for stack creation progress.

## Step 8: Verify the Cluster

Once the cluster shows `Active`:

```bash
# Download kubeconfig from Rancher
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

In the Rancher UI:

- Verify node count matches your configuration
- Check that all system pods are running
- Open the kubectl shell

## Step 9: Post-Provisioning Configuration

### Install Monitoring

Go to **Apps** and install the **Monitoring** chart.

### Configure Storage

EKS provides the EBS CSI driver. Verify it is installed:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

Create a storage class:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### Set Up IAM Roles for Service Accounts (IRSA)

Enable IRSA for fine-grained IAM permissions:

```bash
# This is typically configured during EKS creation
# Verify the OIDC provider exists
aws eks describe-cluster --name <CLUSTER_NAME> --query "cluster.identity.oidc"
```

## Troubleshooting

- **Provisioning fails**: Check the Rancher server logs and the AWS CloudFormation events for error details.
- **Nodes not joining**: Verify the node group security groups allow communication with the EKS control plane.
- **IAM errors**: Ensure the cloud credential has all required permissions.
- **Subnet issues**: Verify subnets have available IP addresses and correct routing.

## Conclusion

Provisioning EKS clusters from Rancher streamlines AWS Kubernetes deployments while maintaining centralized management. Rancher handles the complexity of EKS cluster creation, node group configuration, and agent deployment, giving you a production-ready cluster managed through a single interface alongside your other Kubernetes clusters.
