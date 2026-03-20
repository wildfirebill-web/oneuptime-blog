# How to Use Ansible to Configure IPv4 Firewall Rules with firewalld

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, firewalld, IPv4, Linux, Firewall, Security, Automation

Description: Use Ansible's ansible.posix.firewalld module to manage IPv4 firewall rules on Linux hosts running firewalld, covering zone configuration, service allowances, port rules, and rich rules.

## Introduction

`ansible.posix.firewalld` manages firewalld zones and rules on RHEL, CentOS, Fedora, and similar Linux distributions. It is idempotent and supports immediate and permanent rule changes.

## Basic Service and Port Rules

```yaml
# configure_firewalld.yml
---
- name: Configure IPv4 firewall rules with firewalld
  hosts: linux_servers
  become: yes
  gather_facts: false

  tasks:
    - name: Ensure firewalld is running
      ansible.builtin.service:
        name: firewalld
        state: started
        enabled: yes

    - name: Allow SSH (permanent)
      ansible.posix.firewalld:
        service: ssh
        permanent: yes
        state: enabled
        zone: public

    - name: Allow HTTP and HTTPS
      ansible.posix.firewalld:
        service: "{{ item }}"
        permanent: yes
        state: enabled
        zone: public
      loop:
        - http
        - https

    - name: Open custom port 8080/TCP
      ansible.posix.firewalld:
        port: 8080/tcp
        permanent: yes
        state: enabled
        zone: public

    - name: Reload firewalld to apply permanent rules
      ansible.builtin.command: firewall-cmd --reload
```

## Zone Management

```yaml
    - name: Move interface to trusted zone
      ansible.posix.firewalld:
        zone: trusted
        interface: eth1
        permanent: yes
        state: enabled

    - name: Set default zone to dmz
      ansible.builtin.command: firewall-cmd --set-default-zone=dmz
      changed_when: false
```

## Rich Rules — Block Specific IP

```yaml
    - name: Block specific IPv4 address
      ansible.posix.firewalld:
        rich_rule: "rule family='ipv4' source address='203.0.113.5' drop"
        permanent: yes
        state: enabled
        zone: public

    - name: Allow specific CIDR to reach port 5432
      ansible.posix.firewalld:
        rich_rule: "rule family='ipv4' source address='10.1.0.0/24' port port='5432' protocol='tcp' accept"
        permanent: yes
        state: enabled
        zone: internal
```

## Source Zone Binding (IP-based zone)

```yaml
    - name: Add trusted internal subnet to internal zone
      ansible.posix.firewalld:
        zone: internal
        source: 10.1.0.0/24
        permanent: yes
        state: enabled

    - name: Add untrusted external subnet to drop zone
      ansible.posix.firewalld:
        zone: drop
        source: 192.0.2.0/24
        permanent: yes
        state: enabled
```

## Remove a Rule

```yaml
    - name: Block Telnet
      ansible.posix.firewalld:
        port: 23/tcp
        permanent: yes
        state: disabled
        zone: public
```

## Full Playbook

```bash
ansible-galaxy collection install ansible.posix
ansible-playbook -i inventory.ini configure_firewalld.yml
```

## Conclusion

`ansible.posix.firewalld` provides idempotent firewalld management. Use `permanent: yes` to persist rules across reboots and follow with `firewall-cmd --reload` to activate them immediately. Use rich rules for CIDR-based IPv4 policies and source bindings to assign subnets to appropriate zones without requiring interface reassignment.
