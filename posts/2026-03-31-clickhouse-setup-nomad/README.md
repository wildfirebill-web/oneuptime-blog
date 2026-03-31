# How to Set Up ClickHouse with Nomad

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Nomad, HashiCorp, Container Orchestration, DevOps

Description: Learn how to deploy ClickHouse on HashiCorp Nomad using a job specification with persistent storage, health checks, and Consul service registration.

---

## Why Run ClickHouse on Nomad

Nomad is a lightweight workload orchestrator from HashiCorp that supports Docker, raw executables, and Java workloads in a single scheduler. If your team already runs Consul and Vault, Nomad integrates naturally with both and is simpler to operate than Kubernetes for non-Kubernetes shops.

## Nomad Job Specification

```hcl
job "clickhouse" {
  datacenters = ["dc1"]
  type        = "service"

  group "clickhouse" {
    count = 1

    volume "clickhouse-data" {
      type      = "host"
      read_only = false
      source    = "clickhouse-data"
    }

    network {
      port "http" { static = 8123 }
      port "tcp"  { static = 9000 }
    }

    task "clickhouse-server" {
      driver = "docker"

      config {
        image = "clickhouse/clickhouse-server:24.3"
        ports = ["http", "tcp"]

        ulimit {
          nofile = "262144:262144"
        }
      }

      volume_mount {
        volume      = "clickhouse-data"
        destination = "/var/lib/clickhouse"
        read_only   = false
      }

      resources {
        cpu    = 2000
        memory = 4096
      }

      service {
        name = "clickhouse"
        port = "http"
        tags = ["analytics", "olap"]

        check {
          type     = "http"
          path     = "/ping"
          interval = "10s"
          timeout  = "3s"
        }
      }

      env {
        CLICKHOUSE_DB       = "default"
        CLICKHOUSE_USER     = "default"
        CLICKHOUSE_PASSWORD = ""
      }
    }
  }
}
```

## Host Volume Configuration

Register a host volume on each Nomad client node in `client.hcl`:

```hcl
client {
  host_volume "clickhouse-data" {
    path      = "/data/clickhouse"
    read_only = false
  }
}
```

```bash
mkdir -p /data/clickhouse
chown -R 101:101 /data/clickhouse  # ClickHouse runs as UID 101
```

## Deploying the Job

```bash
nomad job validate clickhouse.nomad
nomad job plan clickhouse.nomad
nomad job run clickhouse.nomad
nomad job status clickhouse
```

## Service Discovery with Consul

The `service` block registers ClickHouse in Consul with a health check. Other services can discover ClickHouse via DNS:

```text
clickhouse.service.consul:8123
```

## Updating the Job

```bash
# Rolling update with no downtime (if running multiple allocations)
nomad job run -check-index $(nomad job inspect clickhouse | jq .JobModifyIndex) clickhouse.nomad
```

## Summary

Running ClickHouse on Nomad requires a job spec with a Docker task, a host volume for data persistence, and a Consul service block for health checking and discovery. Configure the ulimit for open files on the host. Use `nomad job plan` before applying changes to preview the scheduler's diff, and rely on Consul DNS for service-to-service connectivity.
