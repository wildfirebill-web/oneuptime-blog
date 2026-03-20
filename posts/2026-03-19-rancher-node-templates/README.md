# How to Configure Node Templates in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Node Templates, Cloud Credentials

Description: A practical guide to creating and managing node templates in Rancher for standardized node provisioning across cloud providers.

Node templates in Rancher define the machine configuration used when provisioning nodes for your Kubernetes clusters. They specify instance types, disk sizes, operating systems, and cloud-specific settings. This guide covers how to create, manage, and use node templates effectively.

## Prerequisites

- Rancher v2.6 or later
- Cloud credentials configured for your target provider (AWS, Azure, GCP, vSphere, etc.)
- Standard user or admin permissions
- Understanding of your cloud provider's instance types and networking

## What Are Node Templates

Node templates are reusable machine configurations that define the cloud provider settings for cluster nodes. When you create a node pool, you select a node template that determines the instance type, AMI, networking, and storage settings for the nodes in that pool.

## Step 1: Navigate to Node Templates

Access the node template management area:

1. Log in to Rancher.
2. Click on your user avatar in the top-right corner.
3. Select **Node Templates** from the dropdown menu.

You will see a list of existing node templates organized by cloud provider.

## Step 2: Create an AWS Node Template

To create a node template for Amazon EC2:

1. Click **Add Template**.
2. Select **Amazon EC2** as the provider.
3. Choose the cloud credential to use.

Configure the instance settings:

```plaintext
Region: us-east-1
Availability Zone: us-east-1a
VPC/Subnet: Select your target VPC and subnet
Security Group: Select or create a security group
Instance Type: t3.large
Root Disk Size: 100 GB
Root Disk Type: gp3
AMI: ami-0abcdef1234567890 (Ubuntu 22.04)
SSH User: ubuntu
```

Configure IAM settings:

```plaintext
IAM Instance Profile: kubernetes-node-profile
Request Spot Instance: No
```

## Step 3: Create an Azure Node Template

For Azure virtual machines:

1. Click **Add Template**.
2. Select **Azure** as the provider.
3. Choose the cloud credential.

Configure the settings:

```plaintext
Location: eastus
Resource Group: kubernetes-nodes-rg
Availability Set: kubernetes-nodes-as
Image: Canonical:0001-com-ubuntu-server-jammy:22_04-lts:latest
VM Size: Standard_D4s_v3
Storage Type: Premium_LRS
Disk Size: 100 GB
Subnet: kubernetes-subnet
Network Security Group: kubernetes-nsg
SSH User: azureuser
```

## Step 4: Create a vSphere Node Template

For VMware vSphere environments:

1. Click **Add Template**.
2. Select **vSphere** as the provider.
3. Choose the cloud credential.

Configure the virtual machine settings:

```plaintext
Data Center: dc01
Resource Pool: /dc01/host/cluster01/Resources/kubernetes
Datastore: vsanDatastore
Folder: /dc01/vm/kubernetes
Network: VM Network
CPUs: 4
Memory: 8192 MB
Disk Size: 80 GB
Cloud Config:
```

Add cloud-init configuration for vSphere nodes:

```yaml
#cloud-config
package_update: true
packages:
  - curl
  - apt-transport-https
  - ca-certificates
  - software-properties-common
runcmd:
  - sysctl -w net.bridge.bridge-nf-call-iptables=1
  - sysctl -w net.ipv4.ip_forward=1
```

## Step 5: Configure Node Template Labels and Taints

Add Kubernetes labels and taints to your node template:

1. In the node template configuration, scroll to the **Labels** section.
2. Add labels that help identify the node's purpose.

```yaml
# Example labels

node-role.kubernetes.io/worker: "true"
topology.kubernetes.io/zone: us-east-1a
node.kubernetes.io/instance-type: t3.large
team: platform
environment: production
```

Add taints if the nodes should only run specific workloads:

```yaml
# Example taints
- key: dedicated
  value: gpu
  effect: NoSchedule
- key: workload-type
  value: batch
  effect: PreferNoSchedule
```

## Step 6: Set Up Docker Configuration

Configure Docker settings in the node template:

1. Under **Engine Options**, set the Docker version and storage driver.

```plaintext
Docker Version: 24.0.x
Storage Driver: overlay2
Log Driver: json-file
Log Options:
  max-size: 100m
  max-file: "3"
```

Advanced Docker daemon configuration:

```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

## Step 7: Clone and Modify Templates

Save time by cloning existing templates:

1. In the **Node Templates** list, find the template you want to clone.
2. Click the three-dot menu and select **Clone**.
3. Modify the settings as needed (for example, change the instance type or region).
4. Give the new template a descriptive name.

```plaintext
Original: aws-us-east-1-t3-large-production
Clone: aws-us-west-2-t3-large-production
Change: Region from us-east-1 to us-west-2
```

## Step 8: Manage Template Permissions

Control who can use each node template:

1. Node templates are owned by the user who created them by default.
2. To share a template, click the three-dot menu and select **Edit**.
3. Under **Template Access**, change the scope.

Options include:

- **Private**: Only the creator can use this template.
- **Public**: All users in the Rancher installation can use this template.

```bash
# List all node templates via API
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/nodeTemplates" | \
  jq '.data[] | {id: .id, name: .name, driver: .driver, creator: .creatorId}'
```

## Step 9: Use Node Templates in Cluster Creation

Apply node templates when creating clusters:

1. Go to **Cluster Management** and click **Create**.
2. Select your infrastructure provider.
3. In the **Node Pools** section, click **Add Node Pool**.
4. Select the node template for each pool.

```plaintext
Pool 1 - Control Plane:
  Template: aws-controlplane-m5-xlarge
  Count: 3
  Roles: etcd, Control Plane

Pool 2 - Workers:
  Template: aws-worker-t3-large
  Count: 5
  Roles: Worker

Pool 3 - GPU Workers:
  Template: aws-gpu-p3-2xlarge
  Count: 2
  Roles: Worker
```

## Step 10: Clean Up Unused Templates

Periodically review and remove unused node templates:

```bash
# List node templates and their usage
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/nodeTemplates" | \
  jq '.data[] | {name: .name, id: .id, useCount: .useCount}'

# Delete an unused template
curl -s -k \
  -X DELETE \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/nodeTemplates/cattle-global-nt:nt-xxxxx"
```

## Best Practices

- **Name templates descriptively**: Include the provider, region, instance type, and purpose in the template name.
- **Standardize instance types**: Limit the number of different instance types to simplify cost management and capacity planning.
- **Keep AMIs up to date**: Regularly update the base images in your templates to include security patches.
- **Use separate templates per role**: Create distinct templates for control plane, etcd, and worker nodes with appropriate sizing.
- **Test new templates**: Always test a new template by creating a single node before using it in production node pools.

## Conclusion

Node templates in Rancher streamline the process of provisioning consistent and well-configured cluster nodes. By creating standardized templates for different use cases and cloud providers, you reduce manual configuration errors and speed up cluster deployment. Take the time to design your template library thoughtfully, and it will pay dividends as your Kubernetes infrastructure scales.
