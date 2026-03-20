# How to Automate IPv4 Interface Configuration Across Multi-Vendor Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, Multi-Vendor, IPv4, Network Automation, IOS, Junos, EOS, VyOS

Description: Build an Ansible playbook that configures IPv4 interfaces across Cisco IOS, Arista EOS, Juniper JunOS, and VyOS devices using a single role with vendor-specific task files.

## Introduction

Multi-vendor networks require different modules for each OS, but the desired state (IP address, description, no shutdown) is the same. Using Ansible roles with `vars` files and conditional task inclusion, you can manage all vendors from one playbook.

## Directory Structure

```text
ipv4_interfaces/
├── inventory.ini
├── site.yml
├── group_vars/
│   ├── all.yml            # shared vars
│   ├── cisco_ios.yml
│   ├── arista_eos.yml
│   └── juniper_junos.yml
└── tasks/
    ├── ios.yml
    ├── eos.yml
    ├── junos.yml
    └── vyos.yml
```

## group_vars/all.yml

```yaml
interface_configs:
  - interface: "{{ wan_interface }}"
    description: WAN Uplink
    ip: "{{ wan_ip }}"
    prefix: "{{ wan_prefix }}"
  - interface: "{{ lan_interface }}"
    description: LAN Gateway
    ip: "{{ lan_ip }}"
    prefix: 24
```

## inventory.ini

```ini
[cisco_ios]
r1 ansible_host=10.1.1.1 wan_interface=GigabitEthernet0/0 wan_ip=203.0.113.2 wan_prefix=30 lan_interface=GigabitEthernet0/1 lan_ip=192.168.1.1

[arista_eos]
sw1 ansible_host=10.1.1.10 wan_interface=Ethernet1 wan_ip=10.0.0.1 wan_prefix=30 lan_interface=Vlan10 lan_ip=10.1.10.1

[juniper_junos]
jr1 ansible_host=10.1.1.20 wan_interface=ge-0/0/0 wan_ip=203.0.114.2 wan_prefix=30 lan_interface=ge-0/0/1 lan_ip=172.16.1.1

[vyos_routers]
vy1 ansible_host=10.1.1.30 wan_interface=eth0 wan_ip=198.51.100.2 wan_prefix=30 lan_interface=eth1 lan_ip=192.168.2.1
```

## site.yml

```yaml
---
- name: Configure IPv4 interfaces - all vendors
  hosts: all
  gather_facts: false

  tasks:
    - name: Include vendor-specific tasks
      ansible.builtin.include_tasks: "tasks/{{ ansible_network_os }}.yml"
```

## tasks/ios.yml (Cisco IOS)

```yaml
- name: "[IOS] Configure interfaces"
  cisco.ios.ios_config:
    lines:
      - "description {{ item.description }}"
      - "ip address {{ item.ip }} {{ item.prefix | ansible.netcommon.cidr_to_netmask }}"
      - no shutdown
    parents: "interface {{ item.interface }}"
  loop: "{{ interface_configs }}"
```

## tasks/eos.yml (Arista EOS)

```yaml
- name: "[EOS] Configure interfaces"
  arista.eos.eos_config:
    lines:
      - "description {{ item.description }}"
      - "ip address {{ item.ip }}/{{ item.prefix }}"
      - no shutdown
    parents: "interface {{ item.interface }}"
  loop: "{{ interface_configs }}"
```

## tasks/junos.yml (Juniper JunOS)

```yaml
- name: "[JunOS] Configure interfaces"
  junipernetworks.junos.junos_config:
    lines:
      - "set interfaces {{ item.interface }} description '{{ item.description }}'"
      - "set interfaces {{ item.interface }} unit 0 family inet address {{ item.ip }}/{{ item.prefix }}"
  loop: "{{ interface_configs }}"
```

## tasks/vyos.yml (VyOS)

```yaml
- name: "[VyOS] Configure interfaces"
  vyos.vyos.vyos_config:
    lines:
      - "set interfaces ethernet {{ item.interface }} address '{{ item.ip }}/{{ item.prefix }}'"
      - "set interfaces ethernet {{ item.interface }} description '{{ item.description }}'"
  loop: "{{ interface_configs }}"
```

## Run

```bash
ansible-playbook -i inventory.ini site.yml --check --diff
ansible-playbook -i inventory.ini site.yml
```

## Conclusion

Multi-vendor IPv4 automation with Ansible uses `include_tasks` to dispatch to vendor-specific task files based on `ansible_network_os`. A shared data model in `group_vars` holds the desired state; the per-vendor task files translate it to the correct module and syntax. This pattern scales to any number of vendors without modifying the main playbook.
