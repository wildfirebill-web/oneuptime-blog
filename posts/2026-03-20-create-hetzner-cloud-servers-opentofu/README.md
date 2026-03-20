# How to Create Hetzner Cloud Servers with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Hetzner Cloud, Server, Infrastructure as Code, Linux

Description: Learn how to create Hetzner Cloud servers with OpenTofu using the hcloud provider, including SSH keys, user data, and network placement.

Hetzner Cloud offers cost-effective, high-performance cloud servers in Europe and the US. The `hetznercloud/hcloud` OpenTofu provider lets you provision servers, attach them to networks, and configure them with cloud-init as code.

## Provider Configuration

```hcl
terraform {
  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.49"
    }
  }
}

provider "hcloud" {
  # Set via environment variable: export HCLOUD_TOKEN="your-token"
  token = var.hcloud_token
}

variable "hcloud_token" {
  type      = string
  sensitive = true
}
```

## Adding an SSH Key

```hcl
resource "hcloud_ssh_key" "default" {
  name       = "opentofu-key"
  public_key = file("~/.ssh/id_ed25519.pub")
}
```

## Creating a Server

```hcl
resource "hcloud_server" "web" {
  name        = "web-01"
  image       = "ubuntu-24.04"
  server_type = "cx22"      # 2 vCPU, 4 GB RAM
  location    = "nbg1"      # Nuremberg datacenter

  ssh_keys = [hcloud_ssh_key.default.id]

  labels = {
    environment = "production"
    role        = "web"
    managed_by  = "opentofu"
  }
}

output "server_ip" {
  value = hcloud_server.web.ipv4_address
}
```

## Popular Server Types

| Type | vCPU | RAM | Storage |
|---|---|---|---|
| cx22 | 2 | 4 GB | 40 GB |
| cx32 | 4 | 8 GB | 80 GB |
| cx42 | 8 | 16 GB | 160 GB |
| cpx11 | 2 | 2 GB | 40 GB (AMD) |
| cax11 | 2 | 4 GB | 40 GB (ARM64) |

## Using Cloud-Init User Data

```hcl
resource "hcloud_server" "app" {
  name        = "app-01"
  image       = "ubuntu-24.04"
  server_type = "cx32"
  location    = "nbg1"

  ssh_keys = [hcloud_ssh_key.default.id]

  # Cloud-init script to install and configure the application
  user_data = <<-EOT
    #cloud-config
    packages:
      - nginx
      - curl
    runcmd:
      - systemctl enable nginx
      - systemctl start nginx
  EOT
}
```

## Creating Multiple Servers

```hcl
resource "hcloud_server" "workers" {
  count       = 3
  name        = "worker-${count.index + 1}"
  image       = "ubuntu-24.04"
  server_type = "cx22"
  location    = "nbg1"
  ssh_keys    = [hcloud_ssh_key.default.id]

  labels = {
    role  = "worker"
    index = tostring(count.index)
  }
}
```

## Rebuilding a Server Image

```hcl
# Prevent unnecessary rebuilds by ignoring user_data changes after initial creation

resource "hcloud_server" "stable" {
  name        = "stable-app"
  image       = "ubuntu-24.04"
  server_type = "cx22"
  location    = "nbg1"
  ssh_keys    = [hcloud_ssh_key.default.id]
  user_data   = file("cloud-init.yaml")

  lifecycle {
    ignore_changes = [user_data]  # Don't rebuild when user_data changes
  }
}
```

## Conclusion

Creating Hetzner Cloud servers with OpenTofu is straightforward and economical. The `hcloud` provider supports all server types, locations, and cloud-init configuration. Use labels for resource organization, store your API token as a sensitive variable, and use `lifecycle.ignore_changes` for `user_data` if you manage post-provisioning configuration with configuration management tools.
