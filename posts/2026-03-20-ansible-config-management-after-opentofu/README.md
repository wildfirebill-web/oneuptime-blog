# How to Use Ansible for Configuration Management After OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ansible, Infrastructure as Code, Configuration Management, DevOps, Automation

Description: Learn how to integrate Ansible with OpenTofu to provision infrastructure and then apply configuration management in a two-phase pipeline.

---

OpenTofu is ideal for provisioning cloud infrastructure (VMs, networks, databases), while Ansible excels at configuring the software inside those resources. Combining them creates a clean separation: OpenTofu handles "what exists," Ansible handles "what runs on it."

---

## Phase 1: Provision Infrastructure with OpenTofu

```hcl
# main.tf - provision an EC2 instance

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  key_name      = aws_key_pair.deployer.key_name

  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id

  tags = {
    Name = "web-server"
  }
}

# Output the IP for Ansible
output "web_server_ip" {
  value = aws_instance.web.public_ip
}
```

```bash
tofu apply -auto-approve
WEB_IP=$(tofu output -raw web_server_ip)
```

---

## Phase 2: Generate an Ansible Inventory from OpenTofu Output

```bash
# Write a dynamic inventory file
cat > inventory.ini <<EOF
[web]
${WEB_IP} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/deployer.pem
EOF
```

Or use a template file rendered by OpenTofu:

```hcl
resource "local_file" "ansible_inventory" {
  content = templatefile("${path.module}/inventory.tpl", {
    web_ip = aws_instance.web.public_ip
  })
  filename = "${path.module}/inventory.ini"
}
```

---

## Phase 3: Run Ansible Against the Provisioned Infrastructure

```bash
# Wait for SSH to become available
ansible all -i inventory.ini -m ping --timeout=30

# Run a playbook
ansible-playbook -i inventory.ini playbooks/web-server.yml
```

---

## Example Ansible Playbook

```yaml
# playbooks/web-server.yml
- hosts: web
  become: true
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy application config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-enabled/app
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      service:
        name: nginx
        state: reloaded
```

---

## CI/CD Pipeline Integration

```yaml
# .github/workflows/deploy.yml
- name: Provision with OpenTofu
  run: |
    tofu apply -auto-approve
    tofu output -json > tofu-outputs.json

- name: Configure with Ansible
  run: |
    WEB_IP=$(jq -r '.web_server_ip.value' tofu-outputs.json)
    ansible-playbook -i "${WEB_IP}," playbooks/web-server.yml
```

---

## Summary

Use OpenTofu to provision infrastructure and output IP addresses and resource IDs, then feed those values into an Ansible inventory. Run Ansible playbooks to configure the software layer. This two-phase approach keeps infrastructure provisioning and configuration management cleanly separated and composable in CI/CD pipelines.
