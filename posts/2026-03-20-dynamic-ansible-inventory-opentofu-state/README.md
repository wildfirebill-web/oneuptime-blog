# How to Use Dynamic Ansible Inventory from OpenTofu State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ansible, Dynamic Inventory, State, Automation

Description: Learn how to create a dynamic Ansible inventory script that reads directly from OpenTofu state files, ensuring Ansible always targets the current infrastructure without manual inventory updates.

## Introduction

A dynamic inventory script queries OpenTofu state at runtime and returns the current hosts and variables in Ansible's expected JSON format. This approach is more robust than generated static files because it always reflects the live state, not a snapshot from the last `tofu output` run.

## Python Dynamic Inventory Script

```python
#!/usr/bin/env python3
# inventory/opentofu_inventory.py

import json
import subprocess
import sys
import os

def get_tofu_state(working_dir=None):
    """Run tofu show -json and return the parsed state."""
    cmd = ["tofu", "show", "-json"]
    result = subprocess.run(
        cmd,
        capture_output=True,
        text=True,
        cwd=working_dir or os.environ.get("TF_WORKING_DIR", "."),
        check=True
    )
    return json.loads(result.stdout)

def get_tofu_outputs(working_dir=None):
    """Run tofu output -json and return parsed outputs."""
    cmd = ["tofu", "output", "-json"]
    result = subprocess.run(
        cmd,
        capture_output=True,
        text=True,
        cwd=working_dir or os.environ.get("TF_WORKING_DIR", "."),
        check=True
    )
    return json.loads(result.stdout)

def build_inventory(state, outputs):
    """Build Ansible inventory from OpenTofu state and outputs."""
    inventory = {
        "_meta": {"hostvars": {}},
        "all": {"children": [], "vars": {}},
    }

    # Extract EC2 instances from state
    web_hosts = []
    db_hosts = []

    if "values" in state and "root_module" in state["values"]:
        resources = state["values"]["root_module"].get("resources", [])

        for resource in resources:
            if resource["type"] == "aws_instance":
                attrs = resource["values"]
                name = attrs.get("tags", {}).get("Name", resource["address"])
                ip = attrs.get("public_ip") or attrs.get("private_ip")
                role = attrs.get("tags", {}).get("Role", "")

                if ip:
                    inventory["_meta"]["hostvars"][name] = {
                        "ansible_host": ip,
                        "private_ip": attrs.get("private_ip"),
                        "instance_id": attrs.get("id"),
                        "instance_type": attrs.get("instance_type"),
                    }

                    if role == "web":
                        web_hosts.append(name)
                    elif role == "database":
                        db_hosts.append(name)

    # Add groups
    if web_hosts:
        inventory["web"] = {"hosts": web_hosts}
        inventory["all"]["children"].append("web")

    if db_hosts:
        inventory["database"] = {"hosts": db_hosts}
        inventory["all"]["children"].append("database")

    # Add outputs as global vars
    for key, output in outputs.items():
        if not output.get("sensitive"):
            inventory["all"]["vars"][key] = output["value"]

    inventory["all"]["vars"]["ansible_user"] = "ubuntu"
    inventory["all"]["vars"]["ansible_python_interpreter"] = "/usr/bin/python3"

    return inventory

def main():
    if "--list" in sys.argv:
        working_dir = os.environ.get("TF_WORKING_DIR")
        state = get_tofu_state(working_dir)
        outputs = get_tofu_outputs(working_dir)
        inventory = build_inventory(state, outputs)
        print(json.dumps(inventory, indent=2))

    elif "--host" in sys.argv:
        # Individual host vars already included in _meta
        print(json.dumps({}))

if __name__ == "__main__":
    main()
```

## Using the Dynamic Inventory

```bash
# Make the script executable

chmod +x inventory/opentofu_inventory.py

# Set working directory
export TF_WORKING_DIR=/path/to/terraform/root

# List inventory
ansible-inventory -i inventory/opentofu_inventory.py --list

# Run a playbook
ansible-playbook -i inventory/opentofu_inventory.py playbooks/site.yml

# Run ad-hoc command
ansible -i inventory/opentofu_inventory.py web -m ping
```

## Configuring in ansible.cfg

```ini
# ansible.cfg
[defaults]
inventory = ./inventory/opentofu_inventory.py
private_key_file = ~/.ssh/id_rsa
remote_user = ubuntu
host_key_checking = False

[inventory]
# Cache the inventory for 5 minutes to avoid repeated tofu calls
cache = yes
cache_plugin = jsonfile
cache_connection = /tmp/ansible-inventory-cache
cache_timeout = 300
```

## Reading from Remote State via S3

```python
# For reading state directly from S3 without running tofu locally
import boto3

def get_state_from_s3(bucket, key, region="us-east-1"):
    s3 = boto3.client("s3", region_name=region)
    response = s3.get_object(Bucket=bucket, Key=key)
    return json.loads(response["Body"].read())

# Usage in inventory script
state = get_state_from_s3(
    bucket=os.environ["TF_STATE_BUCKET"],
    key=os.environ["TF_STATE_KEY"]
)
```

## Conclusion

A dynamic inventory script that reads OpenTofu state eliminates the gap between infrastructure changes and Ansible's view of the world. The script is most useful in environments where infrastructure changes frequently - new servers are automatically discovered without any manual inventory updates. Cache the inventory with `cache_timeout` to avoid calling `tofu show` on every Ansible task.
