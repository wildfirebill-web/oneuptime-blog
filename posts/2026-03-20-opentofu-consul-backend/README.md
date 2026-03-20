# How to Configure the Consul Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backends

Description: Learn how to configure the Consul backend in OpenTofu to store state in HashiCorp Consul's key-value store with built-in locking.

## Introduction

The Consul backend stores OpenTofu state in HashiCorp Consul's key-value store. It provides built-in state locking via Consul sessions. This is a good choice for teams already running Consul for service discovery or configuration management who want a unified infrastructure backend.

## Basic Configuration

```hcl
terraform {
  backend "consul" {
    address = "consul.acme-corp.com:8500"
    scheme  = "https"
    path    = "terraform/production/state"
  }
}
```

## Authentication with Access Tokens

```hcl
terraform {
  backend "consul" {
    address      = "consul.acme-corp.com:8500"
    scheme       = "https"
    path         = "terraform/production/state"
    access_token = var.consul_token
  }
}
```

Or via environment variable:

```bash
export CONSUL_HTTP_TOKEN="your-consul-token"
export CONSUL_HTTP_ADDR="consul.acme-corp.com:8500"
tofu init
```

## TLS Configuration

```hcl
terraform {
  backend "consul" {
    address    = "consul.acme-corp.com:8501"
    scheme     = "https"
    path       = "terraform/production/state"
    ca_file    = "/etc/ssl/consul/ca.pem"
    cert_file  = "/etc/ssl/consul/client.pem"
    key_file   = "/etc/ssl/consul/client-key.pem"
  }
}
```

## State Locking

The Consul backend uses Consul sessions for locking. The lock is automatically created when a run starts and released when it ends.

```hcl
terraform {
  backend "consul" {
    address = "localhost:8500"
    path    = "terraform/state"
    lock    = true  # Default — enable locking
  }
}
```

## Organization by Path

Use Consul's KV path hierarchy to organize multiple configurations:

```
terraform/
├── networking/state
├── eks/state
├── databases/state
└── apps/
    ├── api/state
    └── frontend/state
```

```hcl
# networking/backend.tf
terraform {
  backend "consul" {
    path = "terraform/networking/state"
  }
}

# eks/backend.tf
terraform {
  backend "consul" {
    path = "terraform/eks/state"
  }
}
```

## Required Consul ACL Policy

```hcl
# Consul ACL policy for OpenTofu state access
key_prefix "terraform/" {
  policy = "write"
}

session_prefix "" {
  policy = "write"
}
```

## Gzip Compression

For large state files, enable gzip compression:

```hcl
terraform {
  backend "consul" {
    address = "consul.acme-corp.com:8500"
    path    = "terraform/production/state"
    gzip    = true
  }
}
```

## Datacenter Configuration

```hcl
terraform {
  backend "consul" {
    address    = "consul.acme-corp.com:8500"
    path       = "terraform/production/state"
    datacenter = "us-east-1"
  }
}
```

## Using Consul Connect

```bash
# Access Consul via a sidecar proxy
export CONSUL_HTTP_ADDR="127.0.0.1:8500"
export CONSUL_HTTP_TOKEN="your-token"
# (Connect proxy runs on localhost:8500)
tofu init
```

## Conclusion

The Consul backend integrates naturally with teams using HashiCorp's stack. State is stored in Consul's KV store with path-based organization and session-based locking. Use ACL policies to restrict access to specific KV prefixes, enable TLS for transport security, and consider gzip compression for large state files.
