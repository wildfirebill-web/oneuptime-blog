# How to Deploy Fluentd Log Collection with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Fluentd, Kubernetes, Logging, Observability, Infrastructure as Code

Description: Learn how to deploy Fluentd log collection on AWS and Kubernetes using OpenTofu to aggregate, transform, and forward logs to centralized destinations like CloudWatch, Elasticsearch, and S3.

Fluentd is an open-source log collector that unifies log data collection across your infrastructure. It supports hundreds of input plugins (containers, files, syslog) and output plugins (Elasticsearch, S3, CloudWatch, Datadog). Managing Fluentd deployments in OpenTofu ensures consistent log collection configuration across environments.

## Deploy Fluentd on Kubernetes with Helm

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
  }
}

provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

# Deploy Fluentd as a DaemonSet (runs on every node)

resource "helm_release" "fluentd" {
  name             = "fluentd"
  repository       = "https://fluent.github.io/helm-charts"
  chart            = "fluentd"
  namespace        = "logging"
  create_namespace = true
  version          = "0.5.2"

  values = [
    yamlencode({
      replicaCount = 1  # DaemonSet manages replicas per node

      env = [
        {
          name  = "ELASTICSEARCH_HOST"
          value = var.elasticsearch_endpoint
        },
        {
          name  = "ELASTICSEARCH_PORT"
          value = "9200"
        }
      ]

      # Use custom fluentd config
      fileConfigs = {
        "01_sources.conf"    = file("${path.module}/fluentd/01_sources.conf")
        "02_filters.conf"    = file("${path.module}/fluentd/02_filters.conf")
        "03_destinations.conf" = file("${path.module}/fluentd/03_destinations.conf")
      }

      resources = {
        requests = { cpu = "200m", memory = "512Mi" }
        limits   = { cpu = "1000m", memory = "1Gi" }
      }

      tolerations = [
        {
          key      = "node-role.kubernetes.io/master"
          operator = "Exists"
          effect   = "NoSchedule"
        }
      ]
    })
  ]
}
```

## Fluentd ConfigMap for CloudWatch Output

```hcl
resource "kubernetes_config_map" "fluentd_config" {
  metadata {
    name      = "fluentd-config"
    namespace = "logging"
  }

  data = {
    "fluent.conf" = <<-EOT
      # Collect all container logs
      <source>
        @type tail
        path /var/log/containers/*.log
        pos_file /var/log/fluentd-containers.log.pos
        tag kubernetes.*
        read_from_head true
        <parse>
          @type json
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </parse>
      </source>

      # Enrich with Kubernetes metadata
      <filter kubernetes.**>
        @type kubernetes_metadata
      </filter>

      # Parse JSON application logs
      <filter kubernetes.**>
        @type parser
        key_name log
        reserve_data true
        remove_key_name_field true
        <parse>
          @type multi_format
          <pattern>
            format json
          </pattern>
          <pattern>
            format none
          </pattern>
        </parse>
      </filter>

      # Route to CloudWatch Logs
      <match kubernetes.**>
        @type cloudwatch_logs
        log_group_name "/kubernetes/${var.cluster_name}"
        log_stream_name_key stream
        auto_create_stream true
        region ${var.region}
        <buffer>
          @type file
          path /var/log/fluentd-cloudwatch
          flush_interval 5s
          retry_max_interval 30s
          chunk_limit_size 2m
          total_limit_size 512m
        </buffer>
      </match>
    EOT
  }
}
```

## IAM Role for Fluentd to Write to CloudWatch

```hcl
# IRSA (IAM Roles for Service Accounts) for Fluentd
resource "aws_iam_role" "fluentd" {
  name = "fluentd-cloudwatch-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:oidc-provider/${var.oidc_provider}"
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${var.oidc_provider}:sub" = "system:serviceaccount:logging:fluentd"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "fluentd_cloudwatch" {
  role = aws_iam_role.fluentd.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Resource = "arn:aws:logs:*:*:*"
      }
    ]
  })
}

resource "kubernetes_service_account" "fluentd" {
  metadata {
    name      = "fluentd"
    namespace = "logging"
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.fluentd.arn
    }
  }
}
```

## Fluentd on EC2 with User Data

```hcl
resource "aws_launch_template" "app_with_fluentd" {
  name          = "app-with-fluentd"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  user_data = base64encode(<<-EOT
    #!/bin/bash
    # Install Fluentd (td-agent)
    curl -fsSL https://toolbelt.treasuredata.com/sh/install-amazon2-td-agent4.sh | sh

    # Install CloudWatch Logs plugin
    td-agent-gem install fluent-plugin-cloudwatch-logs

    # Configure Fluentd
    cat > /etc/td-agent/td-agent.conf <<'CONFIG'
    <source>
      @type tail
      path /var/log/app/*.log
      pos_file /var/log/td-agent/app.log.pos
      tag app.logs
      <parse>
        @type json
      </parse>
    </source>

    <match app.**>
      @type cloudwatch_logs
      log_group_name /app/${var.service_name}
      log_stream_name #{Socket.gethostname}
      auto_create_stream true
      region ${var.region}
    </match>
    CONFIG

    systemctl start td-agent
    systemctl enable td-agent
  EOT
  )
}
```

## Conclusion

Fluentd in OpenTofu provides flexible log collection for both Kubernetes and EC2 environments. Deploy as a DaemonSet on Kubernetes using the Helm provider to collect logs from all nodes, use IRSA to grant minimal CloudWatch permissions without static credentials, and configure buffering to handle network interruptions gracefully. Fluentd's plugin ecosystem supports routing logs to multiple destinations simultaneously - ideal for sending logs to both CloudWatch for alerting and S3 or Elasticsearch for long-term search.
