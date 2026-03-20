# How to Automate IPv4 ACL Deployment Across Multiple Switches with Ansible

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, ACL, IPv4, Network Automation, Cisco IOS, Idempotent, Switches

Description: Learn how to automate the deployment of IPv4 access control lists (ACLs) across multiple switches using Ansible, ensuring consistent security policy enforcement at scale.

---

Manually configuring ACLs on dozens of switches is error-prone. Ansible automates ACL deployment with version-controlled, idempotent playbooks.

## Inventory File

```ini
# /etc/ansible/hosts
[access_switches]
sw-01 ansible_host=10.0.1.1
sw-02 ansible_host=10.0.1.2
sw-03 ansible_host=10.0.1.3

[core_switches]
core-01 ansible_host=10.0.0.1

[all:vars]
ansible_network_os=cisco.ios.ios
ansible_connection=network_cli
ansible_user=admin
ansible_password="{{ vault_switch_password }}"
```

## ACL Definition (host_vars or group_vars)

```yaml
# group_vars/access_switches.yml
acls:
  - name: BLOCK_RFC1918_INBOUND
    acl_type: extended
    aces:
      - sequence: 10
        grant: deny
        protocol: ip
        source:
          address: 10.0.0.0
          wildcard_bits: 0.255.255.255
        destination:
          any: true
        remarks: "Block spoofed RFC1918 from external"
      - sequence: 20
        grant: permit
        protocol: ip
        source:
          any: true
        destination:
          any: true
```

## Playbook: Deploy ACLs

```yaml
---
- name: Deploy IPv4 ACLs to access switches
  hosts: access_switches
  gather_facts: false

  tasks:
    - name: Configure ACLs
      cisco.ios.ios_acls:
        config:
          - afi: ipv4
            acls: "{{ acls }}"
        state: merged

    - name: Apply ACL to interface inbound
      cisco.ios.ios_config:
        lines:
          - ip access-group BLOCK_RFC1918_INBOUND in
        parents: interface GigabitEthernet0/1

    - name: Save configuration
      cisco.ios.ios_config:
        save_when: modified
```

## Verify ACL Deployment

```yaml
    - name: Gather ACL facts
      cisco.ios.ios_acls:
        state: gathered
      register: acl_facts

    - name: Assert ACL exists
      assert:
        that:
          - acl_facts.gathered | selectattr('afi','eq','ipv4') | list | length > 0
        fail_msg: "ACL not found on {{ inventory_hostname }}"
```

## Running the Playbook

```bash
# Dry run (check mode)
ansible-playbook -i hosts deploy-acls.yml --check --diff

# Apply with vault password
ansible-playbook -i hosts deploy-acls.yml --ask-vault-pass

# Target a specific switch
ansible-playbook -i hosts deploy-acls.yml --limit sw-01
```

## Key Takeaways

- Store ACL definitions in `group_vars` or `host_vars` YAML files for easy version control and review.
- Use `--check --diff` to preview changes before applying them to production switches.
- The `cisco.ios.ios_acls` module is idempotent: rerunning the playbook only changes what differs.
- Store switch credentials in Ansible Vault to keep secrets out of plaintext inventory files.
