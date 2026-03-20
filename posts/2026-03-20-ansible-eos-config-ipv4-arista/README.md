# How to Use Ansible eos_config for IPv4 on Arista Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, Arista, EOS, IPv4, Network Automation, eos_config

Description: Use the Ansible arista.eos.eos_config module to configure IPv4 interfaces, routing, and VLANs on Arista EOS switches with idempotent configuration management.

## Introduction

The `arista.eos.eos_config` module manages Arista EOS device configuration via eAPI or SSH. It is idempotent and supports both direct lines and configuration templates.

## Inventory

```ini
[arista_switches]
arista1 ansible_host=192.168.1.20

[arista_switches:vars]
ansible_user=admin
ansible_password=AdminPass
ansible_network_os=eos
ansible_connection=network_cli
ansible_become=yes
ansible_become_method=enable
```

## Configure IPv4 Interface

```yaml
# configure_arista_ipv4.yml
---
- name: Configure IPv4 on Arista switches
  hosts: arista_switches
  gather_facts: false

  tasks:
    - name: Configure Management interface
      arista.eos.eos_config:
        lines:
          - ip address 192.168.1.20/24
        parents: interface Management1

    - name: Configure routed interface (SVI)
      arista.eos.eos_config:
        lines:
          - description Corp LAN
          - ip address 10.1.10.1/24
          - no shutdown
        parents: interface Vlan10

    - name: Configure L3 port (routed port)
      arista.eos.eos_config:
        lines:
          - no switchport
          - ip address 10.0.0.1/30
          - description Uplink to Core
        parents: interface Ethernet1
```

## Configure OSPF

```yaml
    - name: Configure OSPF routing
      arista.eos.eos_config:
        lines:
          - router-id 10.0.0.1
          - network 10.1.10.0/24 area 0.0.0.0
          - network 10.0.0.0/30 area 0.0.0.0
          - passive-interface Vlan10
        parents: router ospf 1
```

## Configure BGP

```yaml
    - name: Configure eBGP
      arista.eos.eos_config:
        lines:
          - router-id 10.0.0.1
          - neighbor 10.0.0.2 remote-as 65000
          - neighbor 10.0.0.2 description Core-Router
          - network 10.1.0.0/16
        parents: router bgp 65001
```

## Backup and Save

```yaml
    - name: Backup before change
      arista.eos.eos_config:
        backup: yes
        backup_options:
          dir_path: ./backups/
          filename: "{{ inventory_hostname }}-{{ ansible_date_time.date }}"
        lines: []

    - name: Save configuration
      arista.eos.eos_config:
        save_when: modified
```

## Run the Playbook

```bash
ansible-galaxy collection install arista.eos
ansible-playbook -i inventory.ini configure_arista_ipv4.yml --check --diff
ansible-playbook -i inventory.ini configure_arista_ipv4.yml
```

## Conclusion

`arista.eos.eos_config` manages Arista EOS IPv4 configuration with the same interface as other Ansible network modules. Use `parents` to scope lines to the correct configuration mode, `save_when: modified` to write to NVRAM only on changes, and `backup: yes` to capture pre-change configuration. Check mode (`--check --diff`) previews changes without applying them.
