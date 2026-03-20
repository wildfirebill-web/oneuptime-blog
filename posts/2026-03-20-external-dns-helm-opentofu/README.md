# How to Deploy External DNS on Kubernetes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, ExternalDNS, DNS, OpenTofu, Helm, Route53, Azure DNS

Description: Learn how to deploy ExternalDNS on Kubernetes using OpenTofu and Helm to automatically manage DNS records for Kubernetes Services and Ingresses.

## Overview

ExternalDNS synchronizes Kubernetes Services and Ingresses with DNS providers like AWS Route53, Azure DNS, Google Cloud DNS, and Cloudflare. OpenTofu deploys ExternalDNS via Helm with the appropriate IAM credentials for your DNS provider.

## Step 1: Deploy ExternalDNS with Helm

```hcl
# main.tf - Deploy ExternalDNS via Helm
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}

resource "helm_release" "external_dns" {
  name             = "external-dns"
  repository       = "https://kubernetes-sigs.github.io/external-dns/"
  chart            = "external-dns"
  version          = "1.14.0"
  namespace        = "external-dns"
  create_namespace = true

  values = [yamlencode({
    provider = "aws"

    aws = {
      region       = "us-east-1"
      zoneType     = "public"
    }

    # Sync policy - sync = create and delete, upsert-only = only create/update
    policy = "sync"

    # Filter by annotation source
    sources = ["ingress", "service"]

    # Domain filters - only manage records in these zones
    domainFilters = ["example.com"]

    # TXT registry to track ownership
    txtOwnerId = "my-cluster"

    # Only process resources with this annotation
    annotationFilter = "external-dns.alpha.kubernetes.io/managed=true"

    serviceAccount = {
      create = true
      name   = "external-dns"
      annotations = {
        "eks.amazonaws.com/role-arn" = aws_iam_role.external_dns.arn
      }
    }

    resources = {
      requests = { cpu = "50m", memory = "64Mi" }
      limits   = { cpu = "100m", memory = "128Mi" }
    }
  })]
}
```

## Step 2: IAM Role for Route53 Access (AWS)

```hcl
# IAM role with IRSA for ExternalDNS to manage Route53
resource "aws_iam_role" "external_dns" {
  name = "external-dns-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.eks.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.oidc_provider}:sub" = "system:serviceaccount:external-dns:external-dns"
          "${local.oidc_provider}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "external_dns" {
  name = "external-dns-policy"
  role = aws_iam_role.external_dns.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["route53:ChangeResourceRecordSets"]
        Resource = ["arn:aws:route53:::hostedzone/*"]
      },
      {
        Effect   = "Allow"
        Action   = ["route53:ListHostedZones", "route53:ListResourceRecordSets"]
        Resource = ["*"]
      }
    ]
  })
}
```

## Step 3: Azure DNS Provider Configuration

```hcl
# ExternalDNS with Azure DNS provider
resource "helm_release" "external_dns_azure" {
  name             = "external-dns"
  repository       = "https://kubernetes-sigs.github.io/external-dns/"
  chart            = "external-dns"
  version          = "1.14.0"
  namespace        = "external-dns"
  create_namespace = true

  values = [yamlencode({
    provider = "azure"

    azure = {
      resourceGroup   = azurerm_resource_group.rg.name
      tenantId        = data.azurerm_client_config.current.tenant_id
      subscriptionId  = data.azurerm_client_config.current.subscription_id
      useManagedIdentityExtension = true
      userAssignedIdentityID = azurerm_user_assigned_identity.external_dns.client_id
    }

    domainFilters   = ["example.com"]
    policy          = "sync"
    txtOwnerId      = "my-cluster"

    podLabels = {
      "azure.workload.identity/use" = "true"
    }

    serviceAccount = {
      annotations = {
        "azure.workload.identity/client-id" = azurerm_user_assigned_identity.external_dns.client_id
      }
    }
  })]
}
```

## Step 4: Annotate Services and Ingresses

```hcl
# Service with ExternalDNS annotation
resource "kubernetes_service" "app" {
  metadata {
    name      = "my-app"
    namespace = "default"
    annotations = {
      "external-dns.alpha.kubernetes.io/managed"  = "true"
      "external-dns.alpha.kubernetes.io/hostname" = "app.example.com"
      "external-dns.alpha.kubernetes.io/ttl"      = "60"
    }
  }

  spec {
    type = "LoadBalancer"
    selector = { app = "my-app" }

    port {
      port        = 80
      target_port = 8080
    }
  }
}
```

## Summary

ExternalDNS deployed with OpenTofu automates DNS record management for Kubernetes workloads. Using IRSA on AWS or Workload Identity on Azure provides secure, credential-free access to DNS APIs. The `sync` policy with domain filters and ownership TXT records ensures ExternalDNS only manages records it created, preventing accidental deletion of manually created DNS entries.
