# How to Deploy Redis with Pulumi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pulumi, Infrastructure as Code, Kubernetes, Cloud

Description: Learn how to deploy Redis with Pulumi using real programming languages - covering Kubernetes StatefulSet deployment, AWS ElastiCache, and secret management.

---

Pulumi lets you define Redis infrastructure using real programming languages like TypeScript, Python, and Go. Unlike YAML-based tools, Pulumi enables loops, conditionals, and abstractions that make Redis infrastructure reusable and maintainable.

## Deploy Redis on Kubernetes with Pulumi (TypeScript)

```bash
npm install @pulumi/kubernetes
```

```typescript
// index.ts
import * as k8s from "@pulumi/kubernetes";
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const redisPassword = config.requireSecret("redisPassword");

// Create a Secret for Redis password
const redisSecret = new k8s.core.v1.Secret("redis-secret", {
    metadata: { name: "redis-secret" },
    stringData: {
        "redis-password": redisPassword,
    },
});

// Deploy Redis StatefulSet
const redisStatefulSet = new k8s.apps.v1.StatefulSet("redis", {
    metadata: { name: "redis" },
    spec: {
        selector: { matchLabels: { app: "redis" } },
        serviceName: "redis",
        replicas: 1,
        template: {
            metadata: { labels: { app: "redis" } },
            spec: {
                containers: [{
                    name: "redis",
                    image: "redis:7-alpine",
                    command: ["redis-server", "--requirepass", "$(REDIS_PASSWORD)"],
                    env: [{
                        name: "REDIS_PASSWORD",
                        valueFrom: {
                            secretKeyRef: {
                                name: redisSecret.metadata.name,
                                key: "redis-password",
                            },
                        },
                    }],
                    ports: [{ containerPort: 6379 }],
                    resources: {
                        requests: { memory: "256Mi", cpu: "100m" },
                        limits:   { memory: "1Gi",  cpu: "500m" },
                    },
                    volumeMounts: [{
                        name: "redis-data",
                        mountPath: "/data",
                    }],
                }],
            },
        },
        volumeClaimTemplates: [{
            metadata: { name: "redis-data" },
            spec: {
                accessModes: ["ReadWriteOnce"],
                resources: { requests: { storage: "10Gi" } },
            },
        }],
    },
});

// Create a Service
const redisService = new k8s.core.v1.Service("redis-service", {
    metadata: { name: "redis" },
    spec: {
        selector: { app: "redis" },
        ports: [{ port: 6379, targetPort: 6379 }],
        clusterIP: "None", // Headless service for StatefulSet
    },
});

export const redisEndpoint = redisService.metadata.name;
```

## Deploy AWS ElastiCache Redis with Pulumi (Python)

```python
# __main__.py
import pulumi
import pulumi_aws as aws

config = pulumi.Config()

# Create a subnet group
subnet_group = aws.elasticache.SubnetGroup("redis-subnet-group",
    description="Redis subnet group",
    subnet_ids=config.require_object("subnetIds")
)

# Create a parameter group for tuning
param_group = aws.elasticache.ParameterGroup("redis-params",
    family="redis7",
    parameters=[
        aws.elasticache.ParameterGroupParameterArgs(
            name="maxmemory-policy",
            value="allkeys-lru"
        ),
        aws.elasticache.ParameterGroupParameterArgs(
            name="timeout",
            value="300"
        ),
    ]
)

# Create the ElastiCache cluster
redis_cluster = aws.elasticache.ReplicationGroup("redis",
    description="Production Redis cluster",
    node_type="cache.r6g.large",
    num_cache_clusters=2,
    parameter_group_name=param_group.name,
    subnet_group_name=subnet_group.name,
    automatic_failover_enabled=True,
    at_rest_encryption_enabled=True,
    transit_encryption_enabled=True,
    auth_token=config.require_secret("redisAuthToken"),
    engine_version="7.0",
    maintenance_window="sun:05:00-sun:06:00",
)

pulumi.export("redis_endpoint", redis_cluster.primary_endpoint_address)
```

## Store Configuration Secrets

```bash
# Set secret values
pulumi config set --secret redisPassword "StrongPassword123!"

# Deploy
pulumi up
```

## Preview and Deploy

```bash
# Preview changes
pulumi preview

# Deploy
pulumi up

# Destroy when done
pulumi destroy
```

## Summary

Pulumi enables Redis infrastructure definition using real programming languages, which allows reusable functions, type checking, and integration with existing code. Deploy Redis on Kubernetes with StatefulSets or on AWS with ElastiCache using the same workflow. Pulumi's secret management keeps Redis passwords out of plaintext configuration files, and its preview mode shows exactly what will change before applying.
