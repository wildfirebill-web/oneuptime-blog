# How to Use depends_on with Outputs in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Outputs, depends_on, Dependencies, Infrastructure as Code, DevOps

Description: A guide to using depends_on in OpenTofu output blocks to ensure proper dependency ordering with side effects.

## Introduction

The `depends_on` argument in output blocks tells OpenTofu that the output value has a dependency on another resource or module, even when there's no direct reference in the value expression. This is useful for ordering guarantees when dealing with resources that have side effects not captured by resource references.

## Why depends_on in Outputs?

```hcl
# Normally, output dependencies are automatically inferred

output "bucket_url" {
  value = "https://${aws_s3_bucket.main.bucket_domain_name}"
  # OpenTofu knows this depends on aws_s3_bucket.main automatically
}

# But sometimes you need explicit dependencies
output "cluster_ready" {
  value = aws_eks_cluster.main.endpoint
  # We need to ensure all node groups are ready before reporting cluster as ready
  depends_on = [
    aws_eks_node_group.workers,
    aws_eks_node_group.system
  ]
}
```

## Common Use Cases

### Ordering with External Systems

```hcl
resource "aws_eks_cluster" "main" {
  name = "production"
  # ...
}

resource "helm_release" "monitoring" {
  name       = "prometheus"
  chart      = "prometheus"
  repository = "https://prometheus-community.github.io/helm-charts"
  namespace  = "monitoring"
  # ...

  depends_on = [aws_eks_cluster.main]
}

output "monitoring_url" {
  description = "URL for the monitoring dashboard (ready after Helm deployment)"
  value       = "https://monitoring.${aws_eks_cluster.main.endpoint}"

  # Ensure monitoring Helm release is deployed before outputting URL
  depends_on = [helm_release.monitoring]
}
```

### Module-Level depends_on

```hcl
module "database" {
  source = "./modules/rds"
  # ...
}

module "migration" {
  source = "./modules/db-migration"
  # ...
  db_endpoint = module.database.endpoint
  depends_on  = [module.database]
}

# Output that requires migration to be complete
output "app_ready" {
  description = "Application is ready when both DB and migrations are complete"
  value       = module.database.endpoint

  # Ensure migration module has run before declaring app ready
  depends_on = [module.migration]
}
```

### Null Resource Dependencies

```hcl
resource "null_resource" "database_setup" {
  triggers = {
    db_id = aws_db_instance.main.id
  }

  provisioner "local-exec" {
    command = "python scripts/setup_database.py ${aws_db_instance.main.endpoint}"
  }
}

output "database_url" {
  description = "Database URL (ready after initial setup script runs)"
  value       = aws_db_instance.main.endpoint

  # Ensure setup script has run before exposing the URL
  depends_on = [null_resource.database_setup]
}
```

## depends_on with Multiple Resources

```hcl
resource "aws_instance" "app" {
  count = 3
  # ...
}

resource "aws_lb_target_group_attachment" "app" {
  count = length(aws_instance.app)
  # ...
}

output "load_balancer_url" {
  description = "Load balancer URL (ready when all instances are registered)"
  value       = aws_lb.main.dns_name

  depends_on = [
    aws_lb_target_group_attachment.app,  # All attachments must complete
    aws_lb_listener.http,                 # Listener must be configured
    aws_lb_listener.https                 # HTTPS listener must be configured
  ]
}
```

## Avoiding Unnecessary depends_on

```hcl
# UNNECESSARY: OpenTofu infers this dependency automatically
output "bad_example" {
  value = aws_instance.web.public_ip  # Reference already creates dependency

  # This is redundant:
  depends_on = [aws_instance.web]  # Not needed
}

# NECESSARY: No direct reference creates the dependency
output "good_example" {
  value = "https://app.example.com"  # Static value, no resource reference

  # depends_on needed to ensure deployment is complete
  depends_on = [
    aws_route53_record.app,
    aws_cloudfront_distribution.main
  ]
}
```

## Conclusion

Using `depends_on` in output blocks ensures that outputs are only exposed after all required resources and side effects are complete. This is particularly important for outputs representing readiness states or URLs that should only be used after supporting infrastructure is fully operational. Use it sparingly - only when OpenTofu cannot automatically infer the dependency from resource references in the output value.
