# How to Configure IPv6 on Network Devices with Ansible

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, IPv6, Network Automation, Cisco IOS, Junos, Network Devices

Description: A guide to configuring IPv6 addresses, routing, and interfaces on network devices (Cisco IOS, Juniper Junos) using Ansible network modules.

Ansible provides dedicated collections for automating network device configuration. This guide shows how to enable IPv6 on Cisco IOS and Juniper Junos devices, including interface addressing, OSPFv3, and BGP neighbors.

## Cisco IOS IPv6 Configuration

### Enable IPv6 on an Interface

```yaml
# cisco-ipv6-interface.yml - Configure IPv6 on Cisco IOS interface

---
- name: Configure IPv6 addresses on Cisco IOS devices
  hosts: cisco_routers
  gather_facts: false

  tasks:
    - name: Enable IPv6 unicast routing globally
      cisco.ios.ios_config:
        lines:
          - "ipv6 unicast-routing"

    - name: Configure IPv6 address on interface GigabitEthernet0/1
      cisco.ios.ios_l3_interfaces:
        config:
          - name: GigabitEthernet0/1
            ipv6:
              - address: "2001:db8:1::1/64"
        state: merged

    - name: Configure link-local address (optional, usually auto-assigned)
      cisco.ios.ios_config:
        lines:
          - "ipv6 address fe80::1 link-local"
        parents: "interface GigabitEthernet0/1"
```

### Configure OSPFv3 for IPv6 Routing

```yaml
# cisco-ospfv3.yml - Configure OSPFv3 on Cisco IOS
---
    - name: Configure OSPFv3 process
      cisco.ios.ios_config:
        lines:
          - "ipv6 router ospf 1"
          - " router-id 1.1.1.1"
          - " log-adjacency-changes detail"

    - name: Enable OSPFv3 on interface
      cisco.ios.ios_config:
        lines:
          - "ipv6 ospf 1 area 0"
        parents: "interface GigabitEthernet0/1"
```

### Configure IPv6 BGP Neighbors

```yaml
# cisco-bgp-ipv6.yml - Add IPv6 BGP neighbor on Cisco IOS
---
    - name: Configure BGP IPv6 address family
      cisco.ios.ios_bgp_address_family:
        config:
          as_number: "65001"
          address_family:
            - afi: "ipv6"
              neighbor:
                - address: "2001:db8:peer::1"
                  remote_as: "65002"
                  activate: true
        state: merged
```

## Juniper Junos IPv6 Configuration

```yaml
# junos-ipv6.yml - Configure IPv6 on Juniper Junos
---
- name: Configure IPv6 on Juniper devices
  hosts: juniper_routers
  gather_facts: false

  tasks:
    - name: Configure IPv6 address on interface ge-0/0/1
      junipernetworks.junos.junos_l3_interfaces:
        config:
          - name: ge-0/0/1
            ipv6:
              - address: "2001:db8:2::1/64"
        state: merged

    - name: Configure IPv6 static route
      junipernetworks.junos.junos_static_routes:
        config:
          - vrf_name: default
            address_families:
              - afi: "ipv6"
                routes:
                  - dest: "::/0"
                    next_hops:
                      - forward_router_address: "2001:db8:1::fe"
        state: merged
```

## Arista EOS IPv6 Configuration

```yaml
# arista-ipv6.yml - Configure IPv6 on Arista EOS
---
- name: Configure IPv6 on Arista EOS
  hosts: arista_switches
  gather_facts: false

  tasks:
    - name: Enable IPv6 routing
      arista.eos.eos_config:
        lines:
          - "ipv6 unicast-routing"

    - name: Configure IPv6 on VLAN interface
      arista.eos.eos_l3_interfaces:
        config:
          - name: Vlan100
            ipv6:
              - address: "2001:db8:vlan100::1/64"
        state: merged
```

## Verify Configuration

```yaml
# verify-ipv6-devices.yml - Verify IPv6 configuration on network devices
---
- name: Verify IPv6 configuration
  hosts: cisco_routers
  gather_facts: false

  tasks:
    - name: Show IPv6 interface brief
      cisco.ios.ios_command:
        commands:
          - "show ipv6 interface brief"
      register: ipv6_interfaces

    - name: Display IPv6 interface summary
      ansible.builtin.debug:
        var: ipv6_interfaces.stdout_lines
```

## Run

```bash
# Install required collections
ansible-galaxy collection install cisco.ios junipernetworks.junos arista.eos

# Apply configuration
ansible-playbook cisco-ipv6-interface.yml -i inventory.ini
ansible-playbook junos-ipv6.yml -i inventory.ini

# Verify
ansible-playbook verify-ipv6-devices.yml -i inventory.ini
```

Ansible's network modules provide a vendor-agnostic way to configure IPv6 across heterogeneous network environments, enabling consistent IPv6 deployment regardless of the underlying network device vendor.
