# How to Configure Ansible AWX/Tower with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, AWX, Ansible Tower, IPv6, Automation Platform, Networking

Description: A guide to configuring Ansible AWX and Ansible Automation Platform (Tower) to manage IPv6 hosts, including inventory configuration and credential setup.

Ansible AWX (the open-source upstream of Ansible Automation Platform/Tower) can manage IPv6 hosts just like IPv4 hosts, with a few configuration considerations around inventory formats, SSH settings, and network connectivity.

## Step 1: Configure AWX to Reach IPv6 Hosts

AWX itself must have IPv6 connectivity to reach managed IPv6 hosts. On a Kubernetes-deployed AWX, ensure the pods have IPv6 addresses:

```bash
# Check if AWX pods have IPv6 addresses
kubectl get pods -n awx -o wide
kubectl exec -n awx <awx-task-pod> -- ip -6 addr show
```

If the cluster is dual-stack, AWX will automatically have IPv6 connectivity.

## Step 2: Add IPv6 Hosts to AWX Inventory

Via the AWX API or UI, create an inventory and add hosts with IPv6 addresses:

```bash
# AWX API: Create an inventory
curl -s -X POST https://awx.example.com/api/v2/inventories/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "IPv6 Production Fleet",
    "description": "All IPv6-enabled production hosts",
    "organization": 1
  }'

# Add a host with IPv6 address
curl -s -X POST https://awx.example.com/api/v2/hosts/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "web-01",
    "inventory": 1,
    "variables": "ansible_host: \"2001:db8::10\"\nansible_ssh_extra_args: \"-6\""
  }'
```

## Step 3: Configure SSH Credentials for IPv6

AWX SSH credentials work the same for IPv6 as IPv4. Create a Machine credential with your SSH private key:

```bash
# Create an SSH machine credential via API
curl -s -X POST https://awx.example.com/api/v2/credentials/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "IPv6 Production SSH Key",
    "credential_type": 1,
    "organization": 1,
    "inputs": {
      "username": "ubuntu",
      "ssh_key_data": "'"$(cat ~/.ssh/id_ed25519)"'"
    }
  }'
```

## Step 4: Configure Job Template with IPv6-Specific Settings

Create a Job Template that forces IPv6 SSH connections via extra variables:

```bash
# Create a Job Template with IPv6 SSH args
curl -s -X POST https://awx.example.com/api/v2/job_templates/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Deploy to IPv6 Fleet",
    "job_type": "run",
    "inventory": 1,
    "project": 1,
    "playbook": "deploy.yml",
    "credential": 1,
    "extra_vars": "ansible_ssh_extra_args: \"-6\""
  }'
```

## Step 5: AWX Instance Group with IPv6 Execution Node

For large IPv6 deployments, dedicate an execution node with IPv6 connectivity:

```yaml
# awx-operator configuration for IPv6 execution node
# In your AWX operator instance configuration
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-production
spec:
  # Configure the execution environment
  execution_environment_pull_policy: IfNotPresent
  # Add custom SSH args for IPv6 connectivity
  extra_settings:
    - setting: AWX_RUNNER_EXTRA_ARGS
      value: "['-6']"
```

## Step 6: Dynamic IPv6 Inventory Script

Use a custom inventory script that populates IPv6 addresses from your CMDB:

```python
#!/usr/bin/env python3
# dynamic-ipv6-inventory.py - Return IPv6 hosts for AWX
import json
import subprocess

def get_ipv6_hosts():
    # In practice, query your CMDB or cloud API
    return {
        "web_servers": {
            "hosts": ["web-01", "web-02"],
            "vars": {
                "ansible_ssh_extra_args": "-6"
            }
        },
        "_meta": {
            "hostvars": {
                "web-01": {"ansible_host": "2001:db8::10"},
                "web-02": {"ansible_host": "2001:db8::11"}
            }
        }
    }

if __name__ == "__main__":
    print(json.dumps(get_ipv6_hosts(), indent=2))
```

## Step 7: Verify IPv6 Connectivity from AWX

```bash
# Launch a quick ping test job from AWX API
curl -s -X POST https://awx.example.com/api/v2/job_templates/<id>/launch/ \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"limit": "web-01", "extra_vars": {"test_only": true}}'
```

AWX/Tower manages IPv6 hosts effectively with proper inventory variable configuration and ensuring the AWX execution plane itself has IPv6 network access to reach managed hosts.
