# How Portainer Nomad Support Worked (Removed in 2.20)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, HashiCorp Nomad, Deprecated, History, Container Orchestration

Description: A historical overview of Portainer's experimental HashiCorp Nomad integration, which was removed in Portainer 2.20 due to limited adoption and maintenance costs.

---

Portainer briefly supported HashiCorp Nomad as a container orchestration backend alongside Docker and Kubernetes. This experimental integration, introduced in Portainer 2.17 and removed in 2.20, offered a GUI for managing Nomad clusters. This post documents what it looked like and why it was removed.

## What Was Nomad?

HashiCorp Nomad is a flexible workload orchestrator that supports Docker containers, raw binaries, Java apps, and VMs. Unlike Kubernetes, Nomad is simpler to operate and supports non-containerized workloads natively.

Key Nomad concepts:

- **Jobs** — the primary unit of work (equivalent to Kubernetes Deployments)
- **Task Groups** — groups of co-located tasks (loosely equivalent to Pods)
- **Tasks** — individual workloads (Docker containers, exec processes, etc.)
- **Allocations** — scheduled instances of a task group on a Nomad client node

## What the Integration Offered

The Portainer Nomad integration allowed operators to:

1. **Connect a Nomad cluster** — by providing the Nomad API endpoint and ACL token
2. **Browse running jobs** — view job status, allocation counts, and task logs
3. **View allocations** — see where tasks were running on which Nomad clients
4. **Deploy Nomad jobs** — submit HCL job specifications from within Portainer
5. **Access task logs** — view stdout/stderr from Nomad-managed tasks

A sample Nomad job definition that could be deployed via Portainer:

```hcl
# nginx.nomad — Nomad HCL job file
job "nginx" {
  datacenters = ["dc1"]
  type        = "service"

  group "web" {
    count = 2

    network {
      port "http" {
        static = 8080
      }
    }

    task "nginx" {
      driver = "docker"

      config {
        image = "nginx:1.25-alpine"
        ports = ["http"]
      }

      resources {
        cpu    = 200    # MHz
        memory = 128    # MB
      }
    }
  }
}
```

## Why It Was Removed

HashiCorp's shift to the Business Source License (BSL) for Nomad in August 2023 was a significant factor. The BSL restricts use of Nomad in competing products, and Portainer (as a management tool) had to evaluate its compatibility. Beyond licensing:

1. **Limited adoption** — few Portainer users were using Nomad versus Docker and Kubernetes
2. **Maintenance burden** — keeping pace with Nomad API changes required significant engineering effort
3. **Incomplete feature parity** — the integration never reached the depth of the Docker/Kubernetes support
4. **Community priority** — engineering resources were redirected toward Docker and Kubernetes improvements

## Migration for Former Nomad Users

If you relied on Portainer for Nomad management, alternatives include:

- **Nomad UI** — the built-in Nomad web interface at `/ui` provides job management, log viewing, and allocation inspection
- **Levant** — a Nomad deployment tool with templating
- **Waypoint** — HashiCorp's application deployment tool with Nomad support
- **Portainer on Kubernetes** — migrating workloads to Kubernetes gives you Portainer's full feature set

## Summary

Portainer's Nomad integration was a genuinely useful experimental feature for the niche of users running Nomad clusters. Its removal in 2.20 reflected the realities of multi-orchestrator support — maintaining quality integrations across multiple backends requires significant resources, and prioritization favored the more widely-used Docker and Kubernetes backends.
