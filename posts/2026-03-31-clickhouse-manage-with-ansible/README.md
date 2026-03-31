# How to Manage ClickHouse with Ansible

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Ansible, Automation, DevOps, Configuration Management

Description: Learn how to install, configure, and manage ClickHouse using Ansible playbooks for repeatable, idempotent deployments across multiple servers.

---

## Why Use Ansible for ClickHouse

Ansible lets you manage ClickHouse installations declaratively across many servers. A well-written playbook handles installation, configuration file templating, user management, and service lifecycle in a single idempotent run.

## Installing ClickHouse with an Ansible Playbook

```yaml
---
- name: Install ClickHouse
  hosts: clickhouse_servers
  become: yes
  tasks:
    - name: Add ClickHouse apt key
      apt_key:
        url: https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key
        state: present

    - name: Add ClickHouse apt repository
      apt_repository:
        repo: "deb https://packages.clickhouse.com/deb lts main"
        state: present
        filename: clickhouse

    - name: Install ClickHouse server and client
      apt:
        name:
          - clickhouse-server
          - clickhouse-client
        state: present
        update_cache: yes

    - name: Start and enable ClickHouse service
      service:
        name: clickhouse-server
        state: started
        enabled: yes
```

## Templating the config.xml

Store ClickHouse configuration as a Jinja2 template and push it with Ansible.

```yaml
    - name: Deploy ClickHouse config
      template:
        src: templates/config.xml.j2
        dest: /etc/clickhouse-server/config.xml
        owner: clickhouse
        group: clickhouse
        mode: '0640'
      notify: Restart ClickHouse
```

A minimal `templates/config.xml.j2`:

```text
<clickhouse>
  <logger>
    <level>{{ clickhouse_log_level | default('warning') }}</level>
    <log>/var/log/clickhouse-server/clickhouse-server.log</log>
    <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
  </logger>
  <max_connections>{{ clickhouse_max_connections | default(4096) }}</max_connections>
  <listen_host>{{ clickhouse_listen_host | default('0.0.0.0') }}</listen_host>
</clickhouse>
```

## Creating ClickHouse Users with Ansible

```yaml
    - name: Create ClickHouse analytics user
      command: >
        clickhouse-client --query
        "CREATE USER IF NOT EXISTS analytics_user
         IDENTIFIED WITH sha256_password BY '{{ clickhouse_analytics_password }}'
         DEFAULT ROLE analytics_role"
      no_log: true
```

## Handlers

```yaml
  handlers:
    - name: Restart ClickHouse
      service:
        name: clickhouse-server
        state: restarted
```

## Inventory Example

```text
[clickhouse_servers]
ch01.example.com
ch02.example.com

[clickhouse_servers:vars]
clickhouse_log_level=warning
clickhouse_max_connections=8192
```

## Running the Playbook

```bash
ansible-playbook -i inventory/hosts.ini clickhouse.yml --ask-vault-pass
```

## Summary

Ansible is an effective tool for managing ClickHouse at scale. Use apt module for installation, Jinja2 templates for config files, the `command` module for DDL operations, and handlers to restart the service only when configuration changes. Store sensitive values like passwords in Ansible Vault to avoid leaking credentials.
