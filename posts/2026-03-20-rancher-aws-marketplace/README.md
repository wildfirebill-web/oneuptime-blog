# How to Use Rancher with AWS Marketplace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, AWS, Marketplace

Description: Learn how to deploy and manage Rancher through AWS Marketplace, including subscription activation, EKS deployment, and billing integration.

## Introduction

AWS Marketplace offers SUSE Rancher as a managed subscription, allowing you to deploy Rancher on EKS with simplified licensing and consolidated billing on your AWS invoice. This guide covers subscribing to Rancher on AWS Marketplace, deploying it on Amazon EKS, and managing the lifecycle of your Marketplace deployment.

## Prerequisites

- An AWS account with billing and Marketplace access
- `aws`, `kubectl`, `helm`, and `eksctl` CLIs installed
- IAM permissions for EKS, EC2, and Marketplace

## Step 1: Subscribe to Rancher on AWS Marketplace

1. Navigate to the [AWS Marketplace Rancher listing](https://aws.amazon.com/marketplace/pp/prodview-rancher).
2. Click **Continue to Subscribe**.
3. Review the pricing (per-cluster or per-node pricing model).
4. Click **Accept Terms** and wait for the subscription to activate (usually immediate).
5. Click **Continue to Configuration** → select software version → **Continue to Launch**.

## Step 2: Create an EKS Cluster for Rancher

```bash
# Create a dedicated EKS cluster for Rancher management

eksctl create cluster \
  --name rancher-management \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name rancher-ng \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 5 \
  --with-oidc \
  --managed

# Update kubeconfig
aws eks update-kubeconfig \
  --name rancher-management \
  --region us-east-1
```

## Step 3: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

kubectl wait --for=condition=Available \
  deployment/cert-manager \
  -n cert-manager \
  --timeout=120s
```

## Step 4: Deploy Rancher from the Marketplace

```bash
# The Marketplace subscription provides a unique Helm chart with billing hooks

# Add the SUSE Rancher Marketplace Helm repository
helm repo add rancher-marketplace \
  https://charts.rancher.com/server-charts/stable
helm repo update

# Install Rancher with the Marketplace-specific values
helm install rancher rancher-marketplace/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com \
  --set replicas=3 \
  --set global.cattle.psp.enabled=false  # PSP removed in K8s 1.25+

# Watch the deployment
kubectl rollout status deployment/rancher -n cattle-system --timeout=10m
```

## Step 5: Configure the AWS Load Balancer

```bash
# EKS uses AWS Load Balancer Controller for ingress
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=rancher-management \
  --set serviceAccount.create=true

# Get the Rancher ingress address
kubectl get ingress -n cattle-system rancher

# Once an address appears, create your DNS record pointing to it
# AWS Route 53:
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "rancher.example.com.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "<alb-dns-name>"}]
      }
    }]
  }'
```

## Step 6: Activate the Marketplace License in Rancher

After deploying via Marketplace, activate your subscription:

1. Log in to Rancher at `https://rancher.example.com`.
2. Navigate to **☰ → Global Settings → Subscription**.
3. The Marketplace subscription token is auto-detected from the cluster's service role.
4. Click **Activate** to link the billing.

## Step 7: Monitor Usage for Billing

```bash
# AWS Marketplace charges by managed node count
# Monitor your billable node count in Rancher
kubectl get nodes --all-namespaces --no-headers | wc -l

# Check the Marketplace billing dashboard
# AWS Console → AWS Marketplace → Manage Subscriptions → Rancher
```

## Step 8: Scaling and Updates

```bash
# Scale the EKS node group for more Rancher capacity
eksctl scale nodegroup \
  --cluster rancher-management \
  --name rancher-ng \
  --nodes 5

# Update Rancher to a new version
helm repo update
helm upgrade rancher rancher-marketplace/rancher \
  --namespace cattle-system \
  --reuse-values \
  --version <new-version>
```

## Conclusion

Deploying Rancher through AWS Marketplace simplifies procurement, licensing, and billing by consolidating Rancher costs on your AWS invoice. The EKS-hosted deployment ensures a managed Kubernetes control plane for Rancher itself, and the Marketplace integration provides a compliance-friendly audit trail of your software subscriptions. This approach is ideal for enterprises that prefer consolidated cloud billing and AWS-native support pathways.
