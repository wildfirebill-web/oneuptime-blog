# How to Set Up EKS with Karpenter Using OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EKS, Karpenter, Node Provisioning, Kubernetes, Infrastructure as Code

Description: Learn how to install and configure Karpenter on EKS using OpenTofu for fast, flexible, and cost-efficient node provisioning based on pod resource requirements.

## Introduction

Karpenter is an open-source Kubernetes node autoprovisioner that launches nodes in response to unschedulable pods. Unlike Cluster Autoscaler, Karpenter provisions nodes directly via EC2 without pre-configured node groups, enabling faster scale-out and better instance type diversification.

## Prerequisites

- OpenTofu v1.6+
- An existing EKS cluster
- Helm provider configured

## Step 1: Create IAM Role for Karpenter Controller

```hcl
# Karpenter controller role using Pod Identity or IRSA
resource "aws_iam_role" "karpenter_controller" {
  name = "KarpenterControllerRole-${var.cluster_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = "sts:AssumeRoleWithWebIdentity"
      Principal = {
        Federated = aws_iam_openid_connect_provider.cluster.arn
      }
      Condition = {
        StringEquals = {
          "${local.oidc_provider}:sub" = "system:serviceaccount:karpenter:karpenter"
          "${local.oidc_provider}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "karpenter_controller" {
  name = "KarpenterControllerPolicy"
  role = aws_iam_role.karpenter_controller.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:CreateFleet", "ec2:CreateLaunchTemplate",
          "ec2:CreateTags", "ec2:DescribeAvailabilityZones",
          "ec2:DescribeImages", "ec2:DescribeInstances",
          "ec2:DescribeInstanceTypes", "ec2:DescribeSubnets",
          "ec2:DescribeSecurityGroups", "ec2:RunInstances",
          "ec2:TerminateInstances", "iam:PassRole",
          "ssm:GetParameter", "eks:DescribeCluster"
        ]
        Resource = "*"
      }
    ]
  })
}
```

## Step 2: Create Node Instance Role

```hcl
# IAM role for EC2 instances provisioned by Karpenter
resource "aws_iam_role" "karpenter_node" {
  name = "KarpenterNodeRole-${var.cluster_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "karpenter_node_policies" {
  for_each = toset([
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  ])

  role       = aws_iam_role.karpenter_node.name
  policy_arn = each.value
}

resource "aws_iam_instance_profile" "karpenter_node" {
  name = "KarpenterNodeInstanceProfile-${var.cluster_name}"
  role = aws_iam_role.karpenter_node.name
}
```

## Step 3: Deploy Karpenter via Helm

```hcl
resource "helm_release" "karpenter" {
  name       = "karpenter"
  repository = "oci://public.ecr.aws/karpenter"
  chart      = "karpenter"
  namespace  = "karpenter"
  version    = "0.37.0"

  create_namespace = true

  set {
    name  = "settings.clusterName"
    value = var.cluster_name
  }

  set {
    name  = "settings.clusterEndpoint"
    value = aws_eks_cluster.main.endpoint
  }

  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.karpenter_controller.arn
  }
}
```

## Step 4: Create NodePool and EC2NodeClass

```hcl
# EC2NodeClass defines AWS-specific node configuration
resource "kubernetes_manifest" "node_class" {
  manifest = {
    apiVersion = "karpenter.k8s.aws/v1beta1"
    kind       = "EC2NodeClass"
    metadata   = { name = "default" }
    spec = {
      amiFamily = "AL2"
      role      = aws_iam_role.karpenter_node.name
      subnetSelectorTerms = [
        { tags = { "kubernetes.io/cluster/${var.cluster_name}" = "shared" } }
      ]
      securityGroupSelectorTerms = [
        { tags = { "kubernetes.io/cluster/${var.cluster_name}" = "owned" } }
      ]
      blockDeviceMappings = [{
        deviceName = "/dev/xvda"
        ebs = { volumeSize = "50Gi", volumeType = "gp3", encrypted = true }
      }]
    }
  }

  depends_on = [helm_release.karpenter]
}

# NodePool defines scheduling constraints and instance requirements
resource "kubernetes_manifest" "node_pool" {
  manifest = {
    apiVersion = "karpenter.sh/v1beta1"
    kind       = "NodePool"
    metadata   = { name = "default" }
    spec = {
      template = {
        spec = {
          nodeClassRef = { name = "default" }
          requirements = [
            { key = "karpenter.sh/capacity-type", operator = "In", values = ["spot", "on-demand"] },
            { key = "kubernetes.io/arch", operator = "In", values = ["amd64"] },
            { key = "karpenter.k8s.aws/instance-category", operator = "In", values = ["c", "m", "r"] },
            { key = "karpenter.k8s.aws/instance-generation", operator = "Gt", values = ["2"] }
          ]
        }
      }
      limits = { cpu = "1000", memory = "1000Gi" }
      disruption = { consolidationPolicy = "WhenUnderutilized", consolidateAfter = "30s" }
    }
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

Karpenter provides faster and more efficient node provisioning than Cluster Autoscaler by launching nodes directly via EC2 APIs without the intermediary of Auto Scaling Groups. Its consolidation feature continuously right-sizes the cluster by replacing underutilized nodes with smaller ones. Use `spot` and `on-demand` capacity types together for optimal cost-availability tradeoffs.
