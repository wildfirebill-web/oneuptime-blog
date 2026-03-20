# How to Use Ansible junos_config for IPv4 on Juniper Devices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, Juniper, JunOS, IPv4, Network Automation, junos_config

Description: Use the Ansible junipernetworks.junos.junos_config module to push IPv4 interface and routing configurations to Juniper JunOS devices using Junos set commands and Jinja2 templates.

## Introduction

The `junipernetworks.junos.junos_config` module configures Juniper JunOS devices via NETCONF. It accepts both Junos set commands and XML-format configuration, with rollback support for safe deployments.

## Inventory

```ini
[juniper_routers]
jrouter1 ansible_host=192.168.1.10

[juniper_routers:vars]
ansible_user=admin
ansible_password=AdminPass
ansible_network_os=junos
ansible_connection=netconf
```

## Configure IPv4 Interface with Set Commands

```yaml
# configure_junos_ipv4.yml
---
- name: Configure IPv4 on Juniper routers
  hosts: juniper_routers
  gather_facts: false

  tasks:
    - name: Set IPv4 address on ge-0/0/1
      junipernetworks.junos.junos_config:
        lines:
          - set interfaces ge-0/0/1 description "LAN Interface"
          - set interfaces ge-0/0/1 unit 0 family inet address 10.1.0.1/24
          - set interfaces ge-0/0/0 description "WAN Uplink"
          - set interfaces ge-0/0/0 unit 0 family inet address 203.0.113.2/30
        comment: "Configure IPv4 interfaces"
```

## Configure Static Route

```yaml
    - name: Configure default route
      junipernetworks.junos.junos_config:
        lines:
          - set routing-options static route 0.0.0.0/0 next-hop 203.0.113.1
        comment: "Set default gateway"
```

## Configure with XML Template

```yaml
    - name: Apply Junos XML configuration
      junipernetworks.junos.junos_config:
        src: templates/junos_interfaces.xml
        src_format: xml
        update: merge
        comment: "Apply interface template"
```

```xml
<!-- templates/junos_interfaces.xml -->
<configuration>
  <interfaces>
    <interface>
      <name>ge-0/0/1</name>
      <unit>
        <name>0</name>
        <family>
          <inet>
            <address>
              <name>10.1.0.1/24</name>
            </address>
          </inet>
        </family>
      </unit>
    </interface>
  </interfaces>
</configuration>
```

## Rollback on Failure

```yaml
    - name: Configure with rollback on error
      junipernetworks.junos.junos_config:
        lines:
          - set routing-options static route 10.2.0.0/16 next-hop 10.1.0.254
        rollback: 0         # Rollback to last committed if error occurs
        confirm: 5          # Auto-rollback after 5 minutes unless confirmed
```

## Backup Running Config

```yaml
    - name: Retrieve and save current config
      junipernetworks.junos.junos_config:
        retrieve: running
        backup: yes
        backup_options:
          dir_path: ./backups/
          filename: "{{ inventory_hostname }}-{{ lookup('pipe', 'date +%Y%m%d') }}"
        lines: []
```

## Run the Playbook

```bash
# Install collection
ansible-galaxy collection install junipernetworks.junos

# Run with check mode
ansible-playbook -i inventory.ini configure_junos_ipv4.yml --check --diff

# Apply
ansible-playbook -i inventory.ini configure_junos_ipv4.yml
```

## Conclusion

`junipernetworks.junos.junos_config` uses NETCONF for reliable JunOS configuration management. Use set commands for simple changes, XML templates for complex configurations, the `confirm` parameter for automatic rollback on commit timeout, and `backup: yes` to preserve pre-change state. Always test with `--check --diff` before production deployment.
