# How to Create Linode Instances with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Linode, Akamai Cloud, Infrastructure as Code, Virtual Machine

Description: Learn how to create Linode (Akamai Cloud) instances with OpenTofu using the linode provider, including disk images, SSH keys, and user data.

Linode (now Akamai Cloud Compute) provides affordable cloud instances. The `linode/linode` OpenTofu provider lets you provision instances, configure them with cloud-init, and manage them as code.

## Provider Configuration

```hcl
terraform {
  required_providers {
    linode = {
      source  = "linode/linode"
      version = "~> 2.0"
    }
  }
}

provider "linode" {
  # Set via environment variable: export LINODE_TOKEN="your-token"
  token = var.linode_token
}

variable "linode_token" {
  type      = string
  sensitive = true
}
```

## Creating a Basic Instance

```hcl
resource "linode_instance" "web" {
  label           = "web-01"
  region          = "us-east"       # Linode region ID
  type            = "g6-nanode-1"   # Instance plan
  image           = "linode/ubuntu24.04"
  root_pass       = var.root_password
  authorized_keys = [var.ssh_public_key]

  tags = ["web", "production"]
}

variable "root_password" {
  type      = string
  sensitive = true
}

output "instance_ip" {
  value = linode_instance.web.ip_address
}
```

## Common Instance Plans

| Plan | vCPU | RAM | Storage |
|---|---|---|---|
| g6-nanode-1 | 1 | 1 GB | 25 GB |
| g6-standard-1 | 1 | 2 GB | 50 GB |
| g6-standard-2 | 2 | 4 GB | 80 GB |
| g6-standard-4 | 4 | 8 GB | 160 GB |

## Using Stackscript for Configuration

```hcl
resource "linode_stackscript" "install_nginx" {
  label       = "install-nginx"
  description = "Install and configure Nginx"
  script      = <<-EOT
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl enable nginx
    systemctl start nginx
  EOT
  images      = ["linode/ubuntu24.04"]
  is_public   = false
}

resource "linode_instance" "web" {
  label           = "web-01"
  region          = "us-east"
  type            = "g6-standard-1"
  image           = "linode/ubuntu24.04"
  root_pass       = var.root_password
  authorized_keys = [var.ssh_public_key]

  stackscript_id = linode_stackscript.install_nginx.id
}
```

## Creating Multiple Instances

```hcl
variable "instance_count" {
  type    = number
  default = 3
}

resource "linode_instance" "workers" {
  count           = var.instance_count
  label           = "worker-${count.index + 1}"
  region          = "us-east"
  type            = "g6-standard-2"
  image           = "linode/ubuntu24.04"
  root_pass       = var.root_password
  authorized_keys = [var.ssh_public_key]

  tags = ["worker", "production"]
}

output "worker_ips" {
  value = linode_instance.workers[*].ip_address
}
```

## Resizing an Instance

Change the `type` attribute and apply:

```hcl
resource "linode_instance" "web" {
  label = "web-01"
  type  = "g6-standard-4"  # Upgraded from g6-standard-2
  # ...
}
```

Resizing requires the instance to be shut down briefly.

## Conclusion

Linode instances are straightforward to provision with OpenTofu. Use `linode_stackscript` for configuration automation, authorized SSH keys for secure access, and the plan `type` attribute for right-sizing. Store sensitive values like `root_pass` as sensitive variables and avoid hardcoding credentials.
