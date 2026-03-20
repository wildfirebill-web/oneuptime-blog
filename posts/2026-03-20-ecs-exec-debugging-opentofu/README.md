# How to Enable ECS Exec for Container Debugging with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ECS, ECS Exec, Debugging, Fargate, Terraform, Containers

Description: Learn how to enable and use ECS Exec with OpenTofu to run interactive commands in running ECS Fargate containers for debugging without exposing SSH ports.

---

ECS Exec allows you to run interactive commands directly inside running ECS containers using AWS Systems Manager Session Manager - no SSH, no bastion hosts, no open ports required. This guide shows how to enable it with OpenTofu and use it for debugging.

---

## How ECS Exec Works

```text
Developer → AWS CLI → SSM Session Manager → ECS Agent → Container
```

ECS Exec uses SSM to create a secure channel to the container without any inbound network access. The ECS task requires an IAM role with SSM permissions.

---

## Step 1: IAM Role with SSM Permissions

```hcl
# iam.tf

resource "aws_iam_role" "ecs_task" {
  name = "ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "ecs_exec" {
  name = "ecs-exec-policy"
  role = aws_iam_role.ecs_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ssmmessages:CreateControlChannel",
          "ssmmessages:CreateDataChannel",
          "ssmmessages:OpenControlChannel",
          "ssmmessages:OpenDataChannel"
        ]
        Resource = "*"
      }
    ]
  })
}
```

---

## Step 2: Enable ECS Exec on the Service

```hcl
# ecs.tf
resource "aws_ecs_cluster" "main" {
  name = "app-cluster"
}

resource "aws_ecs_task_definition" "app" {
  family                   = "app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 256
  memory                   = 512
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = "app"
      image     = "nginx:latest"
      essential = true
      portMappings = [{ containerPort = 80 }]

      # Required: enable pseudo-terminal for interactive sessions
      pseudoTerminal    = true
      linuxParameters = {
        initProcessEnabled = true
      }
    }
  ])
}

resource "aws_ecs_service" "app" {
  name            = "app-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  # Enable ECS Exec
  enable_execute_command = true

  network_configuration {
    subnets          = data.aws_subnets.private.ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }
}
```

---

## Step 3: Execute Commands in the Container

```bash
# Get the task ARN
CLUSTER="app-cluster"
TASK_ARN=$(aws ecs list-tasks --cluster $CLUSTER \
  --service-name app-service \
  --query 'taskArns[0]' --output text)

echo "Task: $TASK_ARN"

# Check if exec is enabled
aws ecs describe-tasks \
  --cluster $CLUSTER \
  --tasks $TASK_ARN \
  --query 'tasks[0].enableExecuteCommand'

# Open interactive shell
aws ecs execute-command \
  --cluster $CLUSTER \
  --task $TASK_ARN \
  --container app \
  --interactive \
  --command "/bin/bash"

# Run a one-off command (non-interactive)
aws ecs execute-command \
  --cluster $CLUSTER \
  --task $TASK_ARN \
  --container app \
  --interactive \
  --command "ps aux"
```

---

## VPC Endpoint for SSM (Private Subnets)

For tasks in private subnets without NAT gateway, add VPC endpoints:

```hcl
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = data.aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = data.aws_subnets.private.ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ssmmessages" {
  vpc_id              = data.aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ssmmessages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = data.aws_subnets.private.ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

---

## Audit Logging for ECS Exec Sessions

```hcl
resource "aws_ecs_cluster" "main" {
  name = "app-cluster"

  configuration {
    execute_command_configuration {
      logging = "OVERRIDE"

      log_configuration {
        cloud_watch_log_group_name = "/ecs/exec-audit"
        s3_bucket_name             = aws_s3_bucket.exec_logs.bucket
        s3_key_prefix              = "exec-logs/"
      }
    }
  }
}

resource "aws_cloudwatch_log_group" "exec_audit" {
  name              = "/ecs/exec-audit"
  retention_in_days = 90
}
```

---

## Troubleshooting

```bash
# Error: "execute command failed... The execute command requires SSM"
# Fix: Ensure task role has ssmmessages:* permissions

# Check SSM agent status in task
aws ecs describe-tasks --cluster app-cluster --tasks $TASK_ARN \
  --query 'tasks[0].containers[0].managedAgents'

# Error: "InvalidParameterException: Container does not support exec"
# Fix: Add initProcessEnabled = true to container linux parameters

# Check ECS Exec is enabled on service
aws ecs describe-services --cluster app-cluster --services app-service \
  --query 'services[0].enableExecuteCommand'
```

---

## Best Practices

1. **Restrict ECS Exec access with IAM** - use condition keys to limit who can exec into containers
2. **Enable audit logging** to S3/CloudWatch for compliance and security review
3. **Use initProcessEnabled** to prevent zombie processes in the container
4. **Disable in production** for highly sensitive workloads - use only for debugging
5. **Use non-interactive commands** (`--command "ps aux"`) for quick diagnostics without shell access

---

## Conclusion

ECS Exec provides a secure, auditability-first approach to container debugging. Enable it in your OpenTofu service configuration, add the required IAM permissions, and use `aws ecs execute-command` to troubleshoot running containers without any network exposure.

---

*Monitor your ECS containers in production with [OneUptime](https://oneuptime.com) - full-stack observability.*
