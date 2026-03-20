# How to Configure AWS Cloud Provider in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, AWS, Cloud Provider

Description: Configure the AWS cloud provider in Rancher-managed clusters to enable native AWS LoadBalancers, EBS volumes, and EC2 node lifecycle management.

## Introduction

The AWS cloud provider integration allows Kubernetes to natively manage AWS resources: provisioning Elastic Load Balancers for Services of type `LoadBalancer`, dynamically provisioning EBS volumes for PersistentVolumeClaims, and automatically managing node lifecycle when EC2 instances are terminated. This guide covers configuring the AWS cloud provider for RKE2 clusters managed by Rancher.

## Prerequisites

- Rancher managing an RKE2 cluster running on AWS EC2
- An IAM role or IAM user with the required permissions
- All EC2 instances tagged with `kubernetes.io/cluster/<cluster-name>: owned`

## Required IAM Permissions

Create an IAM policy with these permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeRegions",
        "ec2:DescribeRouteTables",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeVolumes",
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:CreateTags",
        "elasticloadbalancing:*",
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeLaunchConfigurations"
      ],
      "Resource": "*"
    }
  ]
}
```

## Step 1: Tag EC2 Instances

Every node in the cluster must be tagged:

```bash
# Tag all EC2 instances in the cluster

CLUSTER_NAME="my-rancher-cluster"

for instance_id in $(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=*${CLUSTER_NAME}*" \
  --query "Reservations[*].Instances[*].InstanceId" \
  --output text); do

  aws ec2 create-tags \
    --resources "${instance_id}" \
    --tags \
      "Key=kubernetes.io/cluster/${CLUSTER_NAME},Value=owned" \
      "Key=Name,Value=${CLUSTER_NAME}-node"
done
```

## Step 2: Configure the Cloud Provider in RKE2

Create the cloud provider configuration file on each node:

```bash
# /etc/rancher/rke2/cloud-config.yaml
sudo cat << 'EOF' > /etc/rancher/rke2/cloud-config.yaml
[Global]
zone=us-east-1a
region=us-east-1
vpc=vpc-xxxxxxxx
subnet-id=subnet-xxxxxxxx
role-arn=arn:aws:iam::ACCOUNT_ID:role/RancherNodeRole
kubernetes-cluster-id=my-rancher-cluster

[LoadBalancer]
cross-zone-load-balancing=true

[ServiceOverride "1"]
Service=ec2
Region=us-east-1
URL=https://ec2.us-east-1.amazonaws.com
EOF
```

## Step 3: Configure RKE2 to Use the Cloud Provider

Update the RKE2 server and agent configuration:

```yaml
# /etc/rancher/rke2/config.yaml (server nodes)
cloud-provider-name: aws
cloud-provider-config: /etc/rancher/rke2/cloud-config.yaml
```

```yaml
# /etc/rancher/rke2/config.yaml (agent nodes)
cloud-provider-name: aws
cloud-provider-config: /etc/rancher/rke2/cloud-config.yaml
```

## Step 4: Configure via Rancher UI (Cluster Edit)

1. In Rancher, navigate to **Cluster Management** → select the cluster → **⋮ → Edit Config**.
2. Under **Add-Ons**, find the **Cloud Provider** section.
3. Select **Amazon** from the Cloud Provider dropdown.
4. Paste your `cloud-config.yaml` content into the configuration box.
5. Click **Save**.

## Step 5: Install the AWS Cloud Controller Manager

For RKE2 clusters, install the out-of-tree AWS CCM:

```bash
# Add the AWS CCM Helm chart
helm repo add aws-ccm https://kubernetes.github.io/cloud-provider-aws
helm repo update

# Install the CCM
helm install aws-cloud-controller-manager aws-ccm/aws-cloud-controller-manager \
  --namespace kube-system \
  --set "args[0]=--v=2" \
  --set "args[1]=--cloud-provider=aws" \
  --set "args[2]=--configure-cloud-routes=false"
```

## Step 6: Install the EBS CSI Driver

```bash
# Add the AWS EBS CSI driver
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

# Install the EBS CSI driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=true \
  --set controller.serviceAccount.annotations."eks.amazonaws.com/role-arn"="arn:aws:iam::ACCOUNT:role/EBSCSIRole"
```

## Step 7: Verify the Integration

```bash
# Create a test LoadBalancer service
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: aws-lb-test
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
EOF

# Watch for the external IP (ELB DNS name) to appear
kubectl get service aws-lb-test -w

# Create a test PVC to verify EBS provisioning
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp2   # AWS default StorageClass
  resources:
    requests:
      storage: 5Gi
EOF

kubectl get pvc ebs-test-pvc -w
```

## Conclusion

Configuring the AWS cloud provider in Rancher enables seamless integration with AWS infrastructure services. Once configured, Kubernetes Services of type `LoadBalancer` automatically provision Elastic Load Balancers, and PVCs using the EBS StorageClass automatically provision EBS volumes. Proper IAM role configuration and EC2 instance tagging are the most common prerequisites that, when missed, cause integration failures.
