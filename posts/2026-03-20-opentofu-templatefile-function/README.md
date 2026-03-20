# How to Use the templatefile Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the templatefile function in OpenTofu to render file-based templates with variable substitution for user data scripts and configuration generation.

## Introduction

The `templatefile` function in OpenTofu reads a template file and renders it with a set of provided variables. Templates use the same `${var}` syntax as HCL strings and also support `%{if ...}`, `%{for ...}`, and `%{endif}` directives for conditional and loop logic.

## Syntax

```hcl
templatefile(path, vars)
```

- **path** — path to the template file
- **vars** — a map of variables to pass into the template
- Returns the rendered string

## Basic Examples

Template file (`hello.tpl`):
```
Hello, ${name}! You are in ${environment}.
```

```hcl
output "rendered" {
  value = templatefile("${path.module}/hello.tpl", {
    name        = "Alice"
    environment = "production"
  })
  # Returns: "Hello, Alice! You are in production."
}
```

## Practical Use Cases

### EC2 User Data Script

Template (`userdata.sh.tpl`):
```bash
#!/bin/bash
set -e

# Install and configure for ${environment}
apt-get update -y
apt-get install -y nginx

# Set hostname
hostnamectl set-hostname "${hostname}"

%{ if enable_monitoring ~}
# Install monitoring agent
curl -sL https://monitoring.example.com/install.sh | bash
%{ endif ~}

# Configure application
cat > /etc/app/config.json <<'EOF'
{
  "environment": "${environment}",
  "log_level": "${log_level}",
  "db_host": "${db_host}"
}
EOF
```

```hcl
locals {
  user_data = templatefile("${path.module}/templates/userdata.sh.tpl", {
    environment        = var.environment
    hostname           = "app-server-01"
    enable_monitoring  = var.environment == "prod"
    log_level          = var.environment == "prod" ? "warn" : "debug"
    db_host            = aws_db_instance.main.endpoint
  })
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  user_data     = local.user_data

  tags = {
    Name = "app-server"
  }
}
```

### Nginx Configuration Template

Template (`nginx.conf.tpl`):
```nginx
server {
    listen 80;
    server_name ${domain};

    location / {
        proxy_pass http://${backend_host}:${backend_port};
        proxy_set_header Host $host;
    }

%{ for header in extra_headers ~}
    add_header ${header.name} "${header.value}";
%{ endfor ~}
}
```

```hcl
locals {
  nginx_config = templatefile("${path.module}/templates/nginx.conf.tpl", {
    domain       = var.domain
    backend_host = aws_instance.api.private_ip
    backend_port = 8080
    extra_headers = [
      { name = "X-Frame-Options", value = "DENY" },
      { name = "X-XSS-Protection", value = "1; mode=block" }
    ]
  })
}

resource "kubernetes_config_map" "nginx" {
  metadata {
    name = "nginx-config"
  }

  data = {
    "nginx.conf" = local.nginx_config
  }
}
```

### Cloud-Init YAML Template

Template (`cloud-init.yaml.tpl`):
```yaml
#cloud-config
hostname: ${hostname}
fqdn: ${hostname}.${domain}

packages:
%{ for pkg in packages ~}
  - ${pkg}
%{ endfor ~}

runcmd:
  - echo "Environment: ${environment}" > /etc/environment
```

```hcl
locals {
  cloud_init = templatefile("${path.module}/templates/cloud-init.yaml.tpl", {
    hostname    = "web-01"
    domain      = "example.com"
    environment = var.environment
    packages    = ["nginx", "docker.io", "python3"]
  })
}
```

## Template Directives

### Conditionals

```
%{ if condition ~}
content when true
%{ else ~}
content when false
%{ endif ~}
```

### Loops

```
%{ for item in items ~}
- ${item}
%{ endfor ~}
```

The `~` after directives strips the adjacent newline.

## Step-by-Step Usage

1. Create a `.tpl` file with `${variable}` placeholders.
2. Call `templatefile(path, { var = value })`.
3. Test rendered output with `tofu console`.

## Conclusion

The `templatefile` function is the most powerful string generation tool in OpenTofu. It separates template logic from HCL code, making complex multi-line configurations like user data scripts, nginx configs, and Kubernetes manifests maintainable and reusable.
