# How to Use Ansible nxos_config for IPv4 on Cisco Nexus

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, Cisco, NX-OS, Nexus, IPv4, Network Automation, Nxos_config

Description: Use the Ansible cisco.nxos.nxos_config module to configure IPv4 interfaces, VLANs, and routing on Cisco Nexus switches running NX-OS.

## Introduction

The `cisco.nxos.nxos_config` module manages Cisco Nexus (NX-OS) configuration via NX-API or SSH. It is idempotent and supports lines-based configuration with parent context, similar to ios_config.

## Inventory

```ini
[nexus_switches]
nexus1 ansible_host=192.168.1.30

[nexus_switches:vars]
ansible_user=admin
ansible_password=AdminPass
ansible_network_os=nxos
ansible_connection=network_cli
```

## Configure IPv4 on Nexus Interface

```yaml
# configure_nexus_ipv4.yml

---
- name: Configure IPv4 on Cisco Nexus
  hosts: nexus_switches
  gather_facts: false

  tasks:
    - name: Enable features
      cisco.nxos.nxos_config:
        lines:
          - feature ospf
          - feature bgp
          - feature interface-vlan

    - name: Configure SVI for VLAN 10
      cisco.nxos.nxos_config:
        lines:
          - description Corp LAN
          - ip address 10.1.10.1/24
          - no shutdown
        parents: interface Vlan10

    - name: Configure routed interface
      cisco.nxos.nxos_config:
        lines:
          - no switchport
          - ip address 10.0.0.1/30
          - description Uplink
          - no shutdown
        parents: interface Ethernet1/1

    - name: Create VLAN
      cisco.nxos.nxos_config:
        lines:
          - name Corp
          - state active
        parents: vlan 10
```

## Configure OSPF

```yaml
    - name: Configure OSPF
      cisco.nxos.nxos_config:
        lines:
          - router-id 10.0.0.1
        parents: router ospf MAIN

    - name: Enable OSPF on interface
      cisco.nxos.nxos_config:
        lines:
          - ip router ospf MAIN area 0
        parents: interface Ethernet1/1
```

## Configure HSRPv2

```yaml
    - name: Configure HSRP on SVI
      cisco.nxos.nxos_config:
        lines:
          - hsrp version 2
          - hsrp 1
          - ip 10.1.10.1
          - priority 110
          - preempt
        parents: interface Vlan10
```

## Backup and Save

```yaml
    - name: Backup config
      cisco.nxos.nxos_config:
        backup: yes
        backup_options:
          dir_path: ./backups/
        lines: []

    - name: Save running config
      cisco.nxos.nxos_config:
        save_when: modified
```

## Run the Playbook

```bash
ansible-galaxy collection install cisco.nxos
ansible-playbook -i inventory.ini configure_nexus_ipv4.yml --check --diff
ansible-playbook -i inventory.ini configure_nexus_ipv4.yml
```

## Conclusion

`cisco.nxos.nxos_config` provides idempotent IPv4 configuration for Cisco Nexus running NX-OS. Enable required NX-OS features (interface-vlan, ospf, bgp) before configuring interfaces that depend on them. Use `parents` for VLANs, SVIs, and routing protocol contexts, and always run with `--check --diff` first to preview changes.
