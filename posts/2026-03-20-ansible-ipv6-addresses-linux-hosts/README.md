# How to Configure IPv6 Addresses on Linux Hosts with Ansible

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, IPv6, Linux, Networking, Network Configuration, Automation

Description: A guide to configuring static and dynamic IPv6 addresses on Linux hosts using Ansible, covering both NetworkManager and Netplan-based systems.

Ansible automates IPv6 address configuration across fleets of Linux servers. Whether you use NetworkManager (RHEL/CentOS), Netplan (Ubuntu), or raw ip commands, Ansible provides modules and tasks to configure IPv6 addresses idempotently.

## Playbook Structure

```text
ipv6-config/
├── inventory.ini
├── site.yml
└── roles/
    └── ipv6_address/
        ├── tasks/main.yml
        └── defaults/main.yml
```

## Method 1: Using the `nmcli` Module (RHEL/CentOS)

```yaml
# roles/ipv6_address/tasks/nmcli.yml

---
- name: Configure IPv6 address via NetworkManager
  community.general.nmcli:
    conn_name: "{{ ansible_default_ipv4.interface }}"
    ifname: "{{ ansible_default_ipv4.interface }}"
    type: ethernet
    state: present
    # Add the static IPv6 address and prefix
    ip6: "{{ ipv6_address }}/{{ ipv6_prefix_length }}"
    # Set the IPv6 default gateway
    gw6: "{{ ipv6_gateway }}"
    # Method: manual for static, auto for SLAAC, dhcp for DHCPv6
    method6: manual
    autoconnect: true
  notify: Restart NetworkManager
```

## Method 2: Using Netplan (Ubuntu 20.04+)

```yaml
# roles/ipv6_address/tasks/netplan.yml
---
- name: Write Netplan IPv6 configuration
  ansible.builtin.template:
    src: netplan-ipv6.yaml.j2
    dest: "/etc/netplan/60-ipv6.yaml"
    owner: root
    group: root
    mode: "0600"
  notify: Apply Netplan
```

Template file `netplan-ipv6.yaml.j2`:

```yaml
# templates/netplan-ipv6.yaml.j2 - Netplan static IPv6 configuration
network:
  version: 2
  ethernets:
    {{ ansible_default_ipv4.interface }}:
      addresses:
        - {{ ipv6_address }}/{{ ipv6_prefix_length }}
      routes:
        - to: ::/0
          via: {{ ipv6_gateway }}
      nameservers:
        addresses:
          - 2001:4860:4860::8888
          - 2001:4860:4860::8844
```

## Method 3: Using the `ip` Command (Temporary, Any Distro)

```yaml
# roles/ipv6_address/tasks/ip-command.yml
---
- name: Add IPv6 address to interface (temporary)
  ansible.builtin.command:
    cmd: "ip -6 addr add {{ ipv6_address }}/{{ ipv6_prefix_length }} dev {{ ansible_default_ipv4.interface }}"
  changed_when: true
  ignore_errors: true  # Address may already exist

- name: Add IPv6 default route (temporary)
  ansible.builtin.command:
    cmd: "ip -6 route add default via {{ ipv6_gateway }} dev {{ ansible_default_ipv4.interface }}"
  changed_when: true
  ignore_errors: true
```

## Complete Site Playbook

```yaml
# site.yml - Apply IPv6 configuration to all web servers
---
- name: Configure IPv6 addresses on Linux hosts
  hosts: web_servers
  become: true
  vars:
    ipv6_address: "2001:db8:1::10"
    ipv6_prefix_length: "64"
    ipv6_gateway: "2001:db8:1::1"

  tasks:
    - name: Include Netplan tasks for Ubuntu
      ansible.builtin.include_tasks: roles/ipv6_address/tasks/netplan.yml
      when: ansible_distribution == "Ubuntu"

    - name: Include nmcli tasks for RHEL/CentOS
      ansible.builtin.include_tasks: roles/ipv6_address/tasks/nmcli.yml
      when: ansible_os_family == "RedHat"

    - name: Verify IPv6 address is configured
      ansible.builtin.command:
        cmd: "ip -6 addr show {{ ansible_default_ipv4.interface }}"
      register: ipv6_result
      changed_when: false

    - name: Assert IPv6 address is present
      ansible.builtin.assert:
        that:
          - ipv6_address in ipv6_result.stdout
        fail_msg: "IPv6 address {{ ipv6_address }} was not configured"
        success_msg: "IPv6 address {{ ipv6_address }} is configured"
```

## Inventory

```ini
# inventory.ini
[web_servers]
web-01 ansible_host=192.168.1.10
web-02 ansible_host=192.168.1.11
web-03 ansible_host=192.168.1.12
```

## Run the Playbook

```bash
# Run with check mode first (dry run)
ansible-playbook site.yml -i inventory.ini --check

# Apply the IPv6 configuration
ansible-playbook site.yml -i inventory.ini
```

Ansible's idempotent task model ensures IPv6 addresses are configured consistently across all hosts, whether you're applying initial configuration or verifying existing deployments.
