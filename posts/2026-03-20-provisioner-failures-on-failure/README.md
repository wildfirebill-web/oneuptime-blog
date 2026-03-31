# Handling Provisioner Failures with on_failure in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provisioner, Error Handling, Infrastructure as Code, Terraform

Description: Learn how to use the on_failure setting in OpenTofu provisioners to control behavior when provisioner scripts fail, including continue and fail modes.

## What is on_failure in Provisioners?

When an OpenTofu provisioner (such as `remote-exec` or `local-exec`) encounters an error, the default behavior is to mark the resource as tainted and fail the apply. The `on_failure` setting gives you control over this behavior.

OpenTofu supports two values:

- **`fail`** (default) - The provisioner failure causes the resource to be tainted and the apply to fail
- **`continue`** - The failure is logged as a warning but execution continues

## Default Behavior: fail

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  provisioner "remote-exec" {
    inline = ["sudo apt-get install -y nginx"]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file(var.private_key_path)
      host        = self.public_ip
    }

    on_failure = fail   # Default - resource is tainted on failure
  }
}
```

If the `apt-get` command fails, the instance is tainted. Running `tofu apply` again will destroy and recreate the instance.

## Continue on Failure

Use `on_failure = continue` when the provisioner failure is non-critical:

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y htop",   # Nice to have, not critical
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file(var.private_key_path)
      host        = self.public_ip
    }

    on_failure = continue
  }

  provisioner "remote-exec" {
    inline = [
      "sudo systemctl start myapp",   # This one must succeed
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file(var.private_key_path)
      host        = self.public_ip
    }

    on_failure = fail
  }
}
```

## Destroy-Time Provisioners with on_failure

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  provisioner "remote-exec" {
    when = destroy

    inline = [
      "sudo systemctl stop myapp",
      "/usr/local/bin/drain-connections.sh",
    ]

    on_failure = continue   # Don't block destroy if deregister fails
  }
}
```

This is important for destroy-time provisioners - if the cleanup script fails, you still want the infrastructure to be destroyed.

## local-exec with on_failure

```hcl
resource "aws_route53_record" "app" {
  zone_id = var.zone_id
  name    = "app.example.com"
  type    = "A"
  ttl     = 300
  records = [aws_instance.app.public_ip]
}

resource "null_resource" "notify_slack" {
  triggers = {
    instance_id = aws_instance.app.id
  }

  provisioner "local-exec" {
    command = "curl -s -X POST ${var.slack_webhook} -d '{\"text\": \"New instance deployed\"}'"

    on_failure = continue   # Don't fail deploy if Slack is unreachable
  }
}
```

## When to Use Each Mode

| Scenario | Recommended `on_failure` |
|---|---|
| Critical bootstrapping (installing app) | `fail` |
| Optional monitoring/logging setup | `continue` |
| Notification hooks | `continue` |
| Destroy-time cleanup | `continue` |
| Database migrations | `fail` |
| Health check registration | `continue` |

## Viewing Tainted Resources

```bash
# Show tainted resources

tofu state list | xargs -I{} tofu state show {} | grep -i tainted

# Remove taint manually if you've fixed the issue
tofu untaint aws_instance.web
```

## Best Practices

1. **Default to `fail`** - silent failures are harder to debug than explicit errors
2. **Use `continue` for notify/register steps** that should not block resource creation
3. **Always use `continue` for destroy provisioners** to prevent resource orphaning
4. **Log extensively in provisioner scripts** so failures are diagnosable
5. **Consider alternatives to provisioners** - user data, Packer images, or configuration management tools are more reliable

## Conclusion

The `on_failure` setting in OpenTofu provisioners gives you granular control over error handling in your infrastructure bootstrapping scripts. Use `fail` for critical operations and `continue` for non-essential steps, especially in destroy-time provisioners where blocking the destroy would be worse than skipping the cleanup.
