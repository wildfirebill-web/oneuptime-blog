# How to Use Creation-Time Provisioners in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provisioners, Infrastructure as Code, EC2, remote-exec, local-exec

Description: Learn how creation-time provisioners work in OpenTofu and when they are appropriate alternatives to cloud-init for instance bootstrapping.

---

Creation-time provisioners in OpenTofu run once when a resource is first created. They are explicitly a last-resort tool — the OpenTofu documentation recommends cloud-init or configuration management tools instead. But understanding them is important for legacy code and specific use cases.

---

## local-exec Provisioner

`local-exec` runs a command on the machine running OpenTofu, not on the remote resource:

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> known_hosts.txt"
  }
}
```

---

## remote-exec Provisioner (SSH)

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deployer.key_name

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/deployer.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum install -y nginx",
      "sudo systemctl enable --now nginx",
    ]
  }
}
```

---

## Creation-Only Behavior

By default, provisioners run only at creation time. The resource must be tainted and re-created to run them again:

```bash
# Taint a resource to force re-creation (and re-run provisioners)
tofu taint aws_instance.web
tofu apply
```

---

## Failure Behavior

```hcl
provisioner "remote-exec" {
  on_failure = continue  # Don't fail the apply if provisioner fails
  inline = ["some-optional-setup.sh"]
}
```

Default is `on_failure = fail` — the resource is tainted if the provisioner fails.

---

## When Provisioners Are Acceptable

- Generating a local inventory file after resource creation
- Triggering an external system (webhook, notification) via `local-exec`
- When the resource is in a private network without SSH access is NOT suitable — use cloud-init

---

## Summary

Creation-time provisioners (`local-exec` and `remote-exec`) execute commands when a resource is first created. Use `local-exec` for local operations like writing inventory files. Prefer cloud-init (`user_data`) over `remote-exec` for instance bootstrapping — it's more reliable, doesn't require SSH at apply time, and works with private instances. Provisioners should be the last resort.
