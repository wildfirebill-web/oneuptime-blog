# How to Configure IPv6 SLAAC Settings with Ansible

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, IPv6, SLAAC, sysctl, Networking, Linux

Description: A guide to configuring IPv6 Stateless Address Autoconfiguration (SLAAC) behavior on Linux hosts using Ansible sysctl tasks.

SLAAC allows IPv6 hosts to automatically configure their own addresses using Router Advertisement prefixes. Several sysctl parameters control SLAAC behavior, including whether to accept RAs, whether to generate temporary addresses for privacy, and how to build interface IDs. This guide manages these settings with Ansible.

## Key SLAAC-Related sysctl Parameters

| Parameter | Default | Purpose |
|---|---|---|
| `net.ipv6.conf.all.accept_ra` | 1 | Accept Router Advertisements |
| `net.ipv6.conf.all.autoconf` | 1 | Enable SLAAC address generation |
| `net.ipv6.conf.all.use_tempaddr` | 0 | Use privacy extensions (temporary addresses) |
| `net.ipv6.conf.all.addr_gen_mode` | 0 | Address generation mode (EUI-64, stable, random) |

## Playbook: Configure SLAAC for Client Hosts

```yaml
# configure-slaac-clients.yml - Enable SLAAC on client Linux hosts
---
- name: Configure SLAAC settings on client hosts
  hosts: client_hosts
  become: true

  tasks:
    - name: Enable Router Advertisement acceptance
      ansible.posix.sysctl:
        name: "{{ item }}"
        value: "1"
        state: present
        sysctl_file: /etc/sysctl.d/99-ipv6-slaac.conf
        reload: false
      loop:
        - net.ipv6.conf.all.accept_ra
        - net.ipv6.conf.default.accept_ra
        - net.ipv6.conf.eth0.accept_ra

    - name: Enable SLAAC address autoconfiguration
      ansible.posix.sysctl:
        name: "{{ item }}"
        value: "1"
        state: present
        sysctl_file: /etc/sysctl.d/99-ipv6-slaac.conf
        reload: false
      loop:
        - net.ipv6.conf.all.autoconf
        - net.ipv6.conf.default.autoconf

    - name: Enable privacy extensions (temporary addresses)
      ansible.posix.sysctl:
        name: "{{ item }}"
        value: "2"  # 2 = prefer temporary addresses
        state: present
        sysctl_file: /etc/sysctl.d/99-ipv6-slaac.conf
        reload: false
      loop:
        - net.ipv6.conf.all.use_tempaddr
        - net.ipv6.conf.default.use_tempaddr

    - name: Apply all sysctl settings
      ansible.builtin.command:
        cmd: sysctl --system
      changed_when: false
```

## Playbook: Disable SLAAC on Server/Router Hosts

Servers that route traffic should not use SLAAC — they need static IPv6 addresses:

```yaml
# disable-slaac-servers.yml - Disable SLAAC on server/router hosts
---
- name: Disable SLAAC on server hosts
  hosts: server_hosts
  become: true

  tasks:
    - name: Disable SLAAC and accept_ra (servers use static addressing)
      ansible.posix.sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        sysctl_file: /etc/sysctl.d/99-ipv6-server.conf
        reload: false
      loop:
        - { name: "net.ipv6.conf.all.accept_ra",  value: "0" }
        - { name: "net.ipv6.conf.default.accept_ra", value: "0" }
        - { name: "net.ipv6.conf.all.autoconf", value: "0" }
        - { name: "net.ipv6.conf.default.autoconf", value: "0" }
        - { name: "net.ipv6.conf.all.use_tempaddr", value: "0" }

    - name: Apply sysctl settings
      ansible.builtin.command:
        cmd: sysctl --system
      changed_when: false
```

## Verify SLAAC Is Working

```yaml
# verify-slaac.yml - Check that SLAAC assigned an address
---
- name: Verify SLAAC address assignment
  hosts: client_hosts
  become: false

  tasks:
    - name: Get IPv6 addresses on primary interface
      ansible.builtin.command:
        cmd: "ip -6 addr show {{ ansible_default_ipv4.interface }} scope global"
      register: ipv6_addrs
      changed_when: false

    - name: Display SLAAC-assigned addresses
      ansible.builtin.debug:
        msg: "{{ ipv6_addrs.stdout_lines }}"

    - name: Assert SLAAC address exists
      ansible.builtin.assert:
        that:
          - "'scope global' in ipv6_addrs.stdout"
        fail_msg: "No global IPv6 address found - SLAAC may not be working"
```

## Run

```bash
# Apply SLAAC config to client hosts
ansible-playbook configure-slaac-clients.yml -i inventory.ini

# Disable SLAAC on server hosts
ansible-playbook disable-slaac-servers.yml -i inventory.ini

# Verify SLAAC assignment
ansible-playbook verify-slaac.yml -i inventory.ini
```

Managing SLAAC settings with Ansible ensures each host type — clients, servers, and routers — gets the appropriate SLAAC behavior, balancing automatic address configuration with security and stability requirements.
