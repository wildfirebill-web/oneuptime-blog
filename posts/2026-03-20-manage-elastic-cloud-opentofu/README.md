# How to Manage Elastic Cloud Resources with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Elastic Cloud, Elasticsearch, Kibana, Search

Description: Learn how to manage Elastic Cloud deployments, Elasticsearch clusters, and Kibana spaces using OpenTofu for reproducible search infrastructure.

## Introduction

The Elastic Cloud provider for OpenTofu manages Elastic Cloud deployments including Elasticsearch clusters, Kibana instances, APM servers, and enterprise search. This enables treating your search and observability infrastructure as code.

## Provider Configuration

```hcl
terraform {
  required_providers {
    ec = {
      source  = "elastic/ec"
      version = "~> 0.9"
    }
    elasticstack = {
      source  = "elastic/elasticstack"
      version = "~> 0.11"
    }
  }
}

provider "ec" {
  apikey = var.elastic_cloud_api_key
}
```

## Creating an Elastic Cloud Deployment

```hcl
# Get the latest deployment template for AWS

data "ec_deployment_template" "aws" {
  id     = "aws-storage-optimized"
  region = "us-east-1"
}

resource "ec_deployment" "main" {
  name                   = "prod-search"
  region                 = "us-east-1"
  version                = "8.12.0"
  deployment_template_id = data.ec_deployment_template.aws.id

  elasticsearch {
    hot {
      autoscaling {}
      size        = "4g"
      zone_count  = 2
    }
    warm {
      autoscaling {}
      size       = "2g"
      zone_count = 2
    }
    config {
      plugins = []
    }
  }

  kibana {
    size       = "1g"
    zone_count = 1
  }

  apm {
    size       = "0.5g"
    zone_count = 1
  }

  tags = {
    Environment = "production"
    ManagedBy   = "OpenTofu"
  }
}

output "elasticsearch_endpoint" {
  value = ec_deployment.main.elasticsearch[0].https_endpoint
}

output "kibana_endpoint" {
  value = ec_deployment.main.kibana[0].https_endpoint
}
```

## Managing Elasticsearch Index Templates

```hcl
provider "elasticstack" {
  elasticsearch {
    endpoints = [ec_deployment.main.elasticsearch[0].https_endpoint]
    username  = ec_deployment.main.elasticsearch[0].username
    password  = ec_deployment.main.elasticsearch[0].password
  }
}

resource "elasticstack_elasticsearch_index_template" "logs" {
  name = "app-logs"

  index_patterns = ["app-logs-*"]

  template {
    settings = jsonencode({
      number_of_shards   = 1
      number_of_replicas = 1
      "index.lifecycle.name"      = "logs-policy"
      "index.lifecycle.rollover_alias" = "app-logs"
    })

    mappings = jsonencode({
      properties = {
        "@timestamp" = { type = "date" }
        level        = { type = "keyword" }
        message      = { type = "text" }
        service      = { type = "keyword" }
        trace_id     = { type = "keyword" }
      }
    })
  }

  priority        = 100
  composed_of     = []
}
```

## ILM Policy for Index Lifecycle

```hcl
resource "elasticstack_elasticsearch_index_lifecycle" "logs" {
  name = "logs-policy"

  hot {
    min_age = "0ms"
    set_priority {
      priority = 100
    }
    rollover {
      max_age             = "7d"
      max_primary_shard_size = "50gb"
    }
  }

  warm {
    min_age = "7d"
    set_priority {
      priority = 50
    }
    shrink {
      number_of_shards = 1
    }
    forcemerge {
      max_num_segments = 1
    }
  }

  cold {
    min_age = "30d"
    set_priority {
      priority = 0
    }
  }

  delete {
    min_age = "90d"
    delete {}
  }
}
```

## Kibana Spaces and Saved Objects

```hcl
resource "elasticstack_kibana_space" "engineering" {
  space_id         = "engineering"
  name             = "Engineering"
  description      = "Engineering team workspace"
  disabled_features = ["ml", "canvas"]
}
```

## Conclusion

Managing Elastic Cloud with OpenTofu enables reproducible search infrastructure with consistent index templates, lifecycle policies, and cluster configurations. The two-provider pattern (ec for cluster creation, elasticstack for cluster configuration) mirrors how you'd provision a server with OpenTofu and configure it with Ansible - each tool in its domain.
