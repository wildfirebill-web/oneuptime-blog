# How to Use depends_on for Explicit Dependencies in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, depends_on, Dependencies, HCL, Infrastructure as Code, DevOps

Description: Learn when and how to use the depends_on meta-argument to express explicit resource dependencies that OpenTofu can't infer automatically.

---

OpenTofu automatically infers dependencies between resources when one resource references another's attributes. However, some dependencies are implicit - resources depend on each other through side effects that aren't expressed in attribute references. The `depends_on` meta-argument lets you declare these hidden dependencies explicitly.

---

## Automatic vs Explicit Dependencies

OpenTofu infers these dependencies automatically:
```hcl
resource "aws_instance" "web" {
  subnet_id = aws_subnet.web.id  # ← OpenTofu sees this reference
  # OpenTofu knows: create aws_subnet.web first
}
```

These dependencies require `depends_on`:
```hcl
# IAM policy attachment takes effect asynchronously

# The EC2 instance should wait until the policy is fully attached
resource "aws_instance" "app" {
  depends_on = [aws_iam_role_policy_attachment.app_role]
  # No direct attribute reference, but order matters
}
```

---

## Basic depends_on Usage

```hcl
# Ensure a security group rule exists before the instance
resource "aws_security_group_rule" "allow_web" {
  type        = "ingress"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web.id
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # Wait for the security group rule to be created first
  # Even though we don't reference the rule's ID anywhere
  depends_on = [aws_security_group_rule.allow_web]
}
```

---

## Depending on Multiple Resources

```hcl
resource "aws_ecs_service" "app" {
  name    = "my-app"
  cluster = aws_ecs_cluster.main.id

  depends_on = [
    aws_iam_role_policy_attachment.ecs_task_execution,
    aws_iam_role_policy_attachment.ecs_task_role,
    aws_lb_listener.https,
    aws_cloudwatch_log_group.app
  ]
}
```

---

## depends_on with Modules

You can declare dependencies between modules:

```hcl
module "database" {
  source = "./modules/database"
}

module "application" {
  source = "./modules/application"

  # Application module depends on database module
  # even if it doesn't directly use any of its outputs
  depends_on = [module.database]
}
```

---

## When NOT to Use depends_on

```hcl
# DON'T use depends_on when there's already an attribute reference
resource "aws_instance" "web" {
  subnet_id = aws_subnet.web.id   # reference already creates dependency

  # WRONG: redundant and makes code harder to read
  depends_on = [aws_subnet.web]
}

# DO use depends_on for implicit dependencies
resource "null_resource" "configure" {
  provisioner "local-exec" {
    command = "ansible-playbook -i ${aws_instance.web.public_ip}, site.yml"
  }

  # Wait for the instance profile to be ready before configuring
  depends_on = [aws_iam_instance_profile.web]
}
```

---

## Common Use Cases

| Scenario | Why depends_on is Needed |
|---|---|
| IAM policy attachments | Policies propagate asynchronously |
| DNS records + certificate validation | Certificate needs DNS record to exist |
| S3 bucket policy + Lambda | Lambda needs bucket policy before first trigger |
| Security group rules + instances | Rules may not be active immediately |
| Helm chart + k8s namespace | Namespace must exist before chart installs |

---

## Summary

Use `depends_on` when resources have a real ordering requirement that OpenTofu can't infer from attribute references. Common cases include IAM policy propagation, DNS/certificate validation timing, and any resource that needs another to be fully operational (not just created). Avoid `depends_on` when a regular attribute reference already captures the dependency - it adds verbosity without value.
