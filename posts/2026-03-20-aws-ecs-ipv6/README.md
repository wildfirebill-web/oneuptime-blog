# How to Configure IPv6 for AWS ECS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, IPv6, ECS, Container, Fargate, Task Networking

Description: Enable IPv6 for AWS ECS tasks using dual-stack VPC subnets, configure Fargate tasks with IPv6 addresses, and set up load balancing for IPv6 container traffic.

## Introduction

AWS ECS supports IPv6 for both Fargate and EC2 launch types when deployed in dual-stack or IPv6-only VPC subnets. ECS tasks with `awsvpc` networking mode get their own network interfaces and can receive IPv6 addresses. This enables containers to serve IPv6 clients directly or through an Application Load Balancer.

## ECS Cluster and Task IPv6 Configuration

```bash
# Create ECS cluster (cluster itself is region-level, IPv6 is per-task)

aws ecs create-cluster \
    --cluster-name ipv6-cluster \
    --capacity-providers FARGATE FARGATE_SPOT

# Register a task definition with awsvpc networking
aws ecs register-task-definition \
    --family web-task \
    --network-mode awsvpc \
    --requires-compatibilities FARGATE \
    --cpu 256 \
    --memory 512 \
    --execution-role-arn arn:aws:iam::123:role/ecsTaskExecutionRole \
    --container-definitions '[{
        "name": "web",
        "image": "nginx:latest",
        "portMappings": [
            {"containerPort": 80, "protocol": "tcp"},
            {"containerPort": 443, "protocol": "tcp"}
        ],
        "essential": true
    }]'

# Run task in IPv6-enabled subnet
aws ecs run-task \
    --cluster ipv6-cluster \
    --task-definition web-task \
    --launch-type FARGATE \
    --network-configuration '{
        "awsvpcConfiguration": {
            "subnets": ["subnet-ipv6-enabled"],
            "securityGroups": ["sg-12345678"],
            "assignPublicIp": "ENABLED"
        }
    }'
```

## Terraform ECS with IPv6

```hcl
# ecs_ipv6.tf

resource "aws_ecs_cluster" "main" {
  name = "ipv6-ecs-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "web" {
  family                   = "web"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([{
    name  = "web"
    image = "nginx:latest"
    portMappings = [
      {
        containerPort = 80
        protocol      = "tcp"
      }
    ]
    essential = true
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/web"
        "awslogs-region"        = "us-east-1"
        "awslogs-stream-prefix" = "ecs"
      }
    }
  }])
}

resource "aws_ecs_service" "web" {
  name            = "web-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.web.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets = [
      aws_subnet.private_a.id,  # IPv6-enabled subnets
      aws_subnet.private_b.id,
    ]
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.web.arn
    container_name   = "web"
    container_port   = 80
  }

  depends_on = [aws_lb_listener.http]
}

# Security group for ECS tasks
resource "aws_security_group" "ecs_tasks" {
  vpc_id = aws_vpc.main.id
  name   = "ecs-tasks-sg"

  # Allow traffic from ALB (both IPv4 and IPv6)
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # Allow outbound (IPv4 + IPv6 for updates, AWS APIs)
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
}
```

## Verify ECS Task IPv6 Addresses

```bash
# List tasks in a service
TASK_ARNS=$(aws ecs list-tasks \
    --cluster ipv6-cluster \
    --service-name web-service \
    --query "taskArns" \
    --output text)

# Get network interface details for tasks
for task_arn in $TASK_ARNS; do
    ENI=$(aws ecs describe-tasks \
        --cluster ipv6-cluster \
        --tasks "$task_arn" \
        --query "tasks[0].attachments[?type=='ElasticNetworkInterface'].details[?name=='networkInterfaceId'].value" \
        --output text)

    if [ -n "$ENI" ]; then
        IPV6=$(aws ec2 describe-network-interfaces \
            --network-interface-ids "$ENI" \
            --query "NetworkInterfaces[0].Ipv6Addresses[0].Ipv6Address" \
            --output text)
        echo "Task: $(basename $task_arn) → IPv6: ${IPV6:-none}"
    fi
done
```

## Conclusion

ECS IPv6 requires dual-stack or IPv6-enabled subnets with IPv6 CIDR blocks assigned. Tasks using `awsvpc` network mode get their own ENIs, which can have IPv6 addresses when the subnet auto-assigns them. The ALB in dualstack mode handles IPv6 client connections and forwards to ECS tasks over IPv4 or IPv6 depending on the subnet configuration. Security groups for ECS tasks must include appropriate IPv6 rules for any direct IPv6 traffic.
