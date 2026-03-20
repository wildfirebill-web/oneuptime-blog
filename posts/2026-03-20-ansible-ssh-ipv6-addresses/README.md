# How to Configure Ansible SSH with IPv6 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, IPv6, SSH, Inventory, Connection, Networking

Description: A guide to configuring Ansible to connect to managed hosts via IPv6 SSH addresses, including inventory setup and SSH configuration.

Ansible communicates with managed hosts over SSH by default. Connecting via IPv6 requires correct inventory formatting, SSH client configuration, and sometimes jump host setup. This guide covers everything needed to run Ansible playbooks over IPv6.

## Step 1: IPv6 Addresses in Ansible Inventory

IPv6 addresses must be enclosed in brackets in the inventory file:

```ini
# inventory.ini - Hosts with IPv6 addresses
[web_servers]
# Direct IPv6 address (requires bracket notation)
web-01 ansible_host=[2001:db8::10]

# Or use ansible_host with the IPv6 address
web-02 ansible_host=2001:db8::11

# Full FQDN with IPv6 resolution
web-03 ansible_host=web03.example.com

[web_servers:vars]
# Tell Ansible to use IPv6 explicitly
ansible_ssh_extra_args="-6"
ansible_user=ubuntu
```

## Step 2: YAML Inventory with IPv6

```yaml
# inventory.yaml - YAML inventory with IPv6 hosts
all:
  children:
    web_servers:
      hosts:
        web-01:
          ansible_host: "2001:db8::10"
          ansible_user: ubuntu
          # Force IPv6 for SSH
          ansible_ssh_extra_args: "-6"
        web-02:
          ansible_host: "2001:db8::11"
          ansible_user: ubuntu
          ansible_ssh_extra_args: "-6"
```

## Step 3: Configure ansible.cfg for IPv6

```ini
# ansible.cfg - Global Ansible configuration for IPv6 connections
[defaults]
inventory = inventory.yaml
remote_user = ubuntu
host_key_checking = False

[ssh_connection]
# Force IPv6 for all SSH connections
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s -6
# Increase timeout for IPv6 connections
timeout = 60
```

## Step 4: SSH Client Configuration for IPv6

Add IPv6 host entries to `~/.ssh/config` for easier management:

```
# ~/.ssh/config - SSH client config for IPv6 managed hosts
Host 2001:db8:*
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    # Force IPv6 binding
    AddressFamily inet6

# Named host with IPv6 address
Host web-prod-01
    HostName 2001:db8::10
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

## Step 5: Test the Connection

```bash
# Test SSH connection to an IPv6 host
ansible web-01 -m ping -i inventory.yaml

# If using brackets in the inventory host line
ansible '[2001:db8::10]' -m ping -i inventory.ini -u ubuntu

# Test with verbose output to see the SSH command used
ansible web-01 -m ping -i inventory.yaml -vvv
```

## Step 6: Use IPv6 Jump Hosts (Bastion)

```yaml
# inventory with IPv6 jump host
web-01:
  ansible_host: "2001:db8:internal::10"
  ansible_user: ubuntu
  # Route SSH through an IPv6 jump host
  ansible_ssh_common_args: >-
    -o ProxyJump='ubuntu@[2001:db8:bastion::1]'
    -o StrictHostKeyChecking=no
```

## Step 7: Handle Mixed IPv4/IPv6 Fleets

Use `ansible_host` dynamically from facts or external inventory:

```yaml
# dynamic-ipv6-inventory.yml - Use IPv6 when available, fall back to IPv4
---
- name: Prefer IPv6 connectivity
  hosts: all

  pre_tasks:
    - name: Set ansible_host to IPv6 if available
      ansible.builtin.set_fact:
        ansible_host: "{{ ansible_all_ipv6_addresses[0] }}"
      when: ansible_all_ipv6_addresses | length > 0
```

## Verify Connectivity Across the Fleet

```bash
# Ping all hosts via IPv6
ansible all -m ping -i inventory.yaml

# Show which address Ansible connected via
ansible all -m setup -a "filter=ansible_all_ipv6_addresses" -i inventory.yaml
```

Connecting Ansible to IPv6 hosts is straightforward once the inventory syntax and SSH configuration are correct — using `-6` SSH extra args and bracket notation for addresses ensures reliable IPv6-only management.
