# How to Deploy Fluent Bit Log Collection with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Fluent Bit, Kubernetes, EKS, Logging, Observability, Infrastructure as Code

Description: Learn how to deploy Fluent Bit log collection on Kubernetes and EKS using OpenTofu to efficiently collect, filter, and forward container logs to CloudWatch, Elasticsearch, and other destinations.

Fluent Bit is a lightweight, high-performance log processor designed for Kubernetes environments. Unlike Fluentd, it has minimal memory footprint (under 1MB), making it ideal for running as a DaemonSet on every node. Managing Fluent Bit deployments in OpenTofu ensures consistent, low-overhead log collection across your clusters.

## Deploy Fluent Bit on EKS with Helm

```hcl
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

resource "helm_release" "fluent_bit" {
  name             = "fluent-bit"
  repository       = "https://fluent.github.io/helm-charts"
  chart            = "fluent-bit"
  namespace        = "logging"
  create_namespace = true
  version          = "0.43.0"

  values = [
    yamlencode({
      serviceAccount = {
        create = true
        annotations = {
          "eks.amazonaws.com/role-arn" = aws_iam_role.fluent_bit.arn
        }
      }

      config = {
        service = <<-EOT
          [SERVICE]
              Flush        5
              Daemon       Off
              Log_Level    info
              Parsers_File parsers.conf
              HTTP_Server  On
              HTTP_Listen  0.0.0.0
              HTTP_Port    2020
        EOT

        inputs = <<-EOT
          [INPUT]
              Name              tail
              Path              /var/log/containers/*.log
              multiline.parser  docker, cri
              Tag               kube.*
              Mem_Buf_Limit     5MB
              Skip_Long_Lines   On
        EOT

        filters = <<-EOT
          [FILTER]
              Name                kubernetes
              Match               kube.*
              Kube_URL            https://kubernetes.default.svc:443
              Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
              Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
              Kube_Tag_Prefix     kube.var.log.containers.
              Merge_Log           On
              Keep_Log            Off
              K8S-Logging.Parser  On
              K8S-Logging.Exclude Off

          [FILTER]
              Name  grep
              Match kube.*
              Exclude log ^$
        EOT

        outputs = <<-EOT
          [OUTPUT]
              Name                cloudwatch_logs
              Match               kube.*
              region              ${var.region}
              log_group_name      /aws/eks/${var.cluster_name}/containers
              log_stream_prefix   fluent-bit-
              auto_create_group   true
              log_retention_days  30
        EOT
      }

      resources = {
        requests = { cpu = "50m", memory = "50Mi" }
        limits   = { cpu = "200m", memory = "200Mi" }  # Very lightweight
      }

      tolerations = [
        {
          operator = "Exists"  # Run on ALL nodes including control plane
        }
      ]
    })
  ]
}
```

## IAM Role for Fluent Bit

```hcl
resource "aws_iam_role" "fluent_bit" {
  name = "fluent-bit-eks-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${replace(var.cluster_oidc_issuer_url, "https://", "")}"
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(var.cluster_oidc_issuer_url, "https://", "")}:sub" = "system:serviceaccount:logging:fluent-bit"
          "${replace(var.cluster_oidc_issuer_url, "https://", "")}:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "fluent_bit_cloudwatch" {
  role       = aws_iam_role.fluent_bit.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}
```

## Multi-Output Configuration (CloudWatch + S3)

```hcl
resource "kubernetes_config_map" "fluent_bit_outputs" {
  metadata {
    name      = "fluent-bit-outputs"
    namespace = "logging"
  }

  data = {
    "outputs.conf" = <<-EOT
      # Forward application logs to CloudWatch
      [OUTPUT]
          Name                cloudwatch_logs
          Match               kube.*
          region              ${var.region}
          log_group_name      /aws/eks/${var.cluster_name}/containers
          log_stream_prefix   ${var.environment}-
          auto_create_group   true

      # Forward all logs to S3 for long-term storage
      [OUTPUT]
          Name                s3
          Match               kube.*
          region              ${var.region}
          bucket              ${aws_s3_bucket.logs.id}
          total_file_size     100M
          upload_timeout      10m
          use_put_object      On
          s3_key_format       /%Y/%m/%d/%H/$TAG[1]
          s3_key_format_tag_delimiters  .

      # Forward error logs to OpenSearch
      [OUTPUT]
          Name                es
          Match               kube.*.error
          Host                ${var.opensearch_endpoint}
          Port                443
          TLS                 On
          Index               fluent-bit
          Type                _doc
          AWS_Auth            On
          AWS_Region          ${var.region}
    EOT
  }
}
```

## Namespace-Based Log Routing

```hcl
resource "kubernetes_config_map" "fluent_bit_routing" {
  metadata {
    name      = "fluent-bit-routing"
    namespace = "logging"
  }

  data = {
    "filters.conf" = <<-EOT
      # Tag production namespace logs separately
      [FILTER]
          Name  rewrite_tag
          Match kube.*
          Rule  $kubernetes['namespace_name'] production prod.$TAG false
          Emitter_Name re_emitted_prod

      # Tag staging namespace logs separately
      [FILTER]
          Name  rewrite_tag
          Match kube.*
          Rule  $kubernetes['namespace_name'] staging staging.$TAG false
          Emitter_Name re_emitted_staging

      # Add environment label to all records
      [FILTER]
          Name  record_modifier
          Match kube.*
          Record cluster_name ${var.cluster_name}
          Record environment ${var.environment}
    EOT
  }
}
```

## Conclusion

Fluent Bit on Kubernetes in OpenTofu provides ultra-lightweight, high-performance log collection with minimal resource overhead. Use IRSA for secure AWS credential management, configure multi-output routing to send logs to CloudWatch for alerting and S3 for long-term archival, and use the rewrite_tag filter to route logs from different namespaces to different destinations. Fluent Bit's built-in Kubernetes filter automatically enriches logs with pod labels, namespace, and container metadata.
