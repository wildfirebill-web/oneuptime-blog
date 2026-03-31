# How to Deploy ClickHouse on AWS ECS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, AWS, ECS, Container, Deployment

Description: Deploy ClickHouse on AWS ECS using Fargate or EC2 launch type with persistent EFS storage, task definitions, and ALB integration.

---

AWS ECS provides a managed container orchestration environment for ClickHouse. While Kubernetes is more common for ClickHouse deployments, ECS is a viable option for teams already invested in the AWS ecosystem.

## Choosing a Launch Type

For ClickHouse on ECS, the **EC2 launch type** is strongly preferred over Fargate because:

- Fargate has memory limits (up to 120 GB per task) that may be insufficient for large workloads
- Fargate does not support persistent local storage - all storage is ephemeral
- EC2 launch type allows you to use NVMe instance store or mount EBS volumes

## Task Definition

Create a task definition for ClickHouse:

```json
{
  "family": "clickhouse",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["EC2"],
  "cpu": "8192",
  "memory": "65536",
  "containerDefinitions": [
    {
      "name": "clickhouse",
      "image": "clickhouse/clickhouse-server:24.3",
      "portMappings": [
        {"containerPort": 8123, "protocol": "tcp"},
        {"containerPort": 9000, "protocol": "tcp"}
      ],
      "mountPoints": [
        {
          "sourceVolume": "clickhouse-data",
          "containerPath": "/var/lib/clickhouse"
        },
        {
          "sourceVolume": "clickhouse-config",
          "containerPath": "/etc/clickhouse-server/config.d"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/clickhouse",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "clickhouse"
        }
      }
    }
  ]
}
```

## Persistent Storage with EFS

ClickHouse requires persistent storage. Use Amazon EFS for shared persistent volumes:

```json
"volumes": [
  {
    "name": "clickhouse-data",
    "efsVolumeConfiguration": {
      "fileSystemId": "fs-0123456789abcdef0",
      "rootDirectory": "/clickhouse/data",
      "transitEncryptionEnabled": true
    }
  }
]
```

Note: EFS is slower than EBS for random reads. For production, prefer EC2 instances with local NVMe storage.

## ECS Service Configuration

```bash
aws ecs create-service \
  --cluster clickhouse-cluster \
  --service-name clickhouse \
  --task-definition clickhouse:1 \
  --desired-count 1 \
  --launch-type EC2 \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-def456],assignPublicIp=DISABLED}"
```

## Application Load Balancer for HTTP

Route the HTTP interface (port 8123) through an ALB:

```bash
aws elbv2 create-target-group \
  --name clickhouse-http \
  --protocol HTTP \
  --port 8123 \
  --vpc-id vpc-0abc123 \
  --target-type ip \
  --health-check-path /ping
```

ClickHouse responds to `GET /ping` with `Ok.`, which is perfect for ALB health checks.

## Injecting Configuration via Secrets Manager

Store ClickHouse user passwords in AWS Secrets Manager and inject them as environment variables:

```json
"secrets": [
  {
    "name": "CLICKHOUSE_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:clickhouse/admin-password"
  }
]
```

## Summary

Deploying ClickHouse on AWS ECS works best with the EC2 launch type for access to persistent local storage. Define tasks with appropriate CPU and memory allocations, use EFS for persistent data when local NVMe is not available, configure an ALB for HTTP access using the `/ping` health check endpoint, and inject secrets via AWS Secrets Manager.
