# How to Pass OpenTofu Outputs to Ansible Inventory

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ansible, Inventory, Outputs, Integration

Description: Learn how to use OpenTofu outputs to automatically generate Ansible inventory files, ensuring Ansible always targets the correct hosts that OpenTofu provisioned.

## Introduction

OpenTofu `output` values provide a structured way to pass infrastructure details to Ansible. Using `tofu output` with JSON formatting or inventory templates ensures Ansible always uses the current infrastructure state rather than manually maintained host files.

## Defining Useful Outputs

```hcl
# outputs.tf
output "web_servers" {
  description = "Web server connection details"
  value = {
    for i, instance in aws_instance.web :
    instance.tags["Name"] => {
      ansible_host = instance.public_ip
      private_ip   = instance.private_ip
      instance_id  = instance.id
      az           = instance.availability_zone
    }
  }
}

output "db_endpoint" {
  description = "Database endpoint for Ansible configuration"
  value       = aws_db_instance.main.endpoint
  sensitive   = false
}

output "redis_endpoint" {
  description = "Redis endpoint for application config"
  value       = aws_elasticache_replication_group.main.primary_endpoint_address
}
```

## INI Format Inventory via Template

```hcl
output "ansible_ini_inventory" {
  value = templatefile("${path.module}/templates/inventory.ini.tpl", {
    web_servers = aws_instance.web[*]
    db_servers  = aws_instance.db[*]
    bastion_ip  = aws_instance.bastion.public_ip
    environment = var.environment
  })
}
```

```ini
# templates/inventory.ini.tpl
[web]
%{ for server in web_servers ~}
${server.tags["Name"]} ansible_host=${server.public_ip} private_ip=${server.private_ip}
%{ endfor ~}

[database]
%{ for server in db_servers ~}
${server.tags["Name"]} ansible_host=${server.private_ip}
%{ endfor ~}

[all:vars]
ansible_user=ubuntu
ansible_ssh_common_args='-o StrictHostKeyChecking=no -J ubuntu@${bastion_ip}'
environment=${environment}
```

## JSON Format Inventory via Template

```hcl
output "ansible_json_inventory" {
  value = jsonencode({
    all = {
      children = {
        web = {
          hosts = {
            for instance in aws_instance.web :
            instance.tags["Name"] => {
              ansible_host = instance.public_ip
              private_ip   = instance.private_ip
            }
          }
        }
        database = {
          hosts = {
            for instance in aws_instance.db :
            instance.tags["Name"] => {
              ansible_host = instance.private_ip
            }
          }
        }
      }
      vars = {
        ansible_user        = "ubuntu"
        ansible_python_interpreter = "/usr/bin/python3"
        db_endpoint         = aws_db_instance.main.endpoint
        redis_endpoint      = aws_elasticache_replication_group.main.primary_endpoint_address
        environment         = var.environment
      }
    }
  })
}
```

## Generating the Inventory File in CI/CD

```bash
#!/bin/bash
# scripts/generate-inventory.sh

# Get OpenTofu outputs
INVENTORY_JSON=$(tofu output -raw ansible_json_inventory)

# Write to inventory file
echo "$INVENTORY_JSON" > inventory/hosts.json

# Verify the inventory
ansible-inventory -i inventory/hosts.json --list

# Run playbook with generated inventory
ansible-playbook -i inventory/hosts.json playbooks/site.yml
```

## Using tofu output Directly

```bash
# Write INI inventory directly from template output
tofu output -raw ansible_ini_inventory > /tmp/inventory.ini

# Or use individual outputs as extra-vars
DB_ENDPOINT=$(tofu output -raw db_endpoint)
REDIS_ENDPOINT=$(tofu output -raw redis_endpoint)

ansible-playbook site.yml \
  -e "db_endpoint=${DB_ENDPOINT}" \
  -e "redis_endpoint=${REDIS_ENDPOINT}"
```

## Makefile Integration

```makefile
# Makefile
.PHONY: provision configure deploy

provision:
	cd infrastructure && tofu apply -auto-approve

inventory: provision
	cd infrastructure && tofu output -raw ansible_ini_inventory > ../ansible/inventory/hosts.ini

configure: inventory
	cd ansible && ansible-playbook -i inventory/hosts.ini playbooks/site.yml

deploy: configure
	@echo "Full deployment complete"
```

## Conclusion

The cleanest way to pass OpenTofu outputs to Ansible is through structured JSON outputs or `templatefile`-generated INI files. This ensures the inventory always reflects the current state of the infrastructure, eliminates manual host file maintenance, and makes it easy to inject infrastructure details like database endpoints and Redis URLs directly into the Ansible run.
