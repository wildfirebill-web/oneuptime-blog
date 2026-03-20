# How to Use Heredoc Syntax for Multi-Line Strings in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, HCL, Heredoc, Multi-Line Strings, Expressions, Infrastructure as Code, DevOps

Description: A guide to using heredoc syntax in OpenTofu to define multi-line string values cleanly in your HCL configurations.

## Introduction

Heredoc syntax in OpenTofu allows you to write multi-line strings without needing to escape newlines or use string concatenation. This is especially useful for inline scripts, JSON documents, YAML content, and any other multi-line text content in your configuration.

## Basic Heredoc Syntax

```hcl
# Standard heredoc: <<IDENTIFIER ... IDENTIFIER
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  user_data = <<EOF
#!/bin/bash
apt-get update
apt-get install -y nginx
systemctl enable nginx
systemctl start nginx
EOF
}
```

## Indented Heredoc

```hcl
# Indented heredoc: <<-IDENTIFIER strips leading whitespace
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    # This script installs and configures the app
    export APP_VERSION="${var.app_version}"
    export APP_ENV="${var.environment}"

    apt-get update
    apt-get install -y python3 python3-pip

    pip3 install myapp==${var.app_version}
    systemctl enable myapp
    systemctl start myapp
  EOF
  # Leading whitespace (4 spaces) is stripped from each line
}
```

## String Interpolation in Heredoc

```hcl
variable "cluster_endpoint" {
  type = string
}

variable "cluster_ca" {
  type = string
}

resource "aws_launch_template" "node" {
  name_prefix   = "eks-node-"
  image_id      = var.ami_id
  instance_type = "t3.medium"

  user_data = base64encode(<<-USERDATA
    #!/bin/bash
    set -ex

    # Bootstrap the node
    /etc/eks/bootstrap.sh ${var.cluster_name} \
      --b64-cluster-ca ${var.cluster_ca} \
      --apiserver-endpoint ${var.cluster_endpoint} \
      --kubelet-extra-args '--node-labels=role=worker'
  USERDATA
  )
}
```

## JSON in Heredoc

```hcl
resource "aws_iam_role" "lambda" {
  name = "lambda-execution-role"

  assume_role_policy = <<-JSON
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "lambda.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }
  JSON
}
```

## YAML in Heredoc

```hcl
resource "kubernetes_config_map" "app" {
  metadata {
    name      = "app-config"
    namespace = "default"
  }

  data = {
    "config.yaml" = <<-YAML
      app:
        name: ${var.app_name}
        version: ${var.app_version}
        environment: ${var.environment}
      database:
        host: ${aws_db_instance.main.address}
        port: ${aws_db_instance.main.port}
        name: ${var.db_name}
      logging:
        level: ${var.environment == "prod" ? "warn" : "debug"}
    YAML
  }
}
```

## Multi-Line Scripts with Heredoc

```hcl
resource "null_resource" "setup" {
  provisioner "local-exec" {
    command = <<-SCRIPT
      #!/bin/bash
      set -euo pipefail

      echo "Setting up environment..."

      # Configure kubectl
      aws eks update-kubeconfig \
        --name ${var.cluster_name} \
        --region ${var.region}

      # Apply base manifests
      kubectl apply -f manifests/namespaces.yaml
      kubectl apply -f manifests/rbac.yaml

      # Install Helm charts
      helm repo add stable https://charts.helm.sh/stable
      helm repo update

      helm upgrade --install \
        ingress-nginx stable/nginx-ingress \
        --namespace kube-system \
        --set controller.replicaCount=2

      echo "Setup complete!"
    SCRIPT
  }
}
```

## Heredoc in Variables

```hcl
variable "nginx_config" {
  type    = string
  default = <<-EOF
    server {
      listen 80;
      server_name _;

      location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }
  EOF
}
```

## Heredoc in Locals

```hcl
locals {
  cloud_init_config = <<-YAML
    #cloud-config
    package_update: true
    packages:
      - nginx
      - python3
    runcmd:
      - systemctl enable nginx
      - systemctl start nginx
      - echo "Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)" > /var/www/html/index.html
  YAML
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  user_data     = local.cloud_init_config
}
```

## Template Functions with Heredoc

```hcl
# Use templatefile() for complex templating (preferred over heredoc with lots of logic)
resource "aws_instance" "app" {
  ami       = var.ami_id
  user_data = templatefile("${path.module}/templates/userdata.sh.tpl", {
    app_version = var.app_version
    environment = var.environment
    db_host     = aws_db_instance.main.address
  })
}

# For simpler cases, heredoc with interpolation works well
locals {
  simple_script = <<-EOF
    #!/bin/bash
    export APP_VERSION="${var.app_version}"
    systemctl restart myapp
  EOF
}
```

## Conclusion

Heredoc syntax significantly improves the readability of OpenTofu configurations that include multi-line strings. Use the indented heredoc (`<<-`) for configurations embedded inside resource blocks to keep indentation clean. Standard interpolation (`${}`) works inside heredoc strings for dynamic values. For complex templates with conditional logic or loops, consider using the `templatefile()` function with separate template files instead. Heredoc is ideal for user data scripts, inline policies, configuration files, and any other multi-line content that needs to be embedded directly in your HCL.
