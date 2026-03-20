# How to Manage Portainer with Ansible

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Ansible, Automation, DevOps, Infrastructure as Code

Description: Use Ansible to automate Portainer management tasks including environment registration, stack deployments, and user management via the Portainer API.

## Introduction

Ansible is a powerful automation tool that can orchestrate your entire Portainer infrastructure. By combining Ansible's idempotent task execution with Portainer's REST API, you can automate environment registration, stack deployments, user creation, and configuration management across all your Portainer instances. This approach is particularly valuable in large-scale deployments where manual UI operations don't scale.

## Prerequisites

- Ansible 2.12+ installed on your control machine
- Portainer instance accessible via HTTP/HTTPS API
- Python 3.8+ with `requests` library
- Portainer admin credentials

## Step 1: Install Required Ansible Collections

```bash
# Install required collections
ansible-galaxy collection install community.docker
ansible-galaxy collection install ansible.builtin

# Install Python dependencies
pip install requests urllib3
```

## Step 2: Set Up Inventory and Variables

Create your Ansible inventory structure:

```ini
# inventory/hosts
[portainer_servers]
portainer-primary ansible_host=192.168.1.100
portainer-secondary ansible_host=192.168.1.101

[portainer_servers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
portainer_port=9443
portainer_protocol=https
```

```yaml
# group_vars/portainer_servers.yml
portainer_admin_user: admin
portainer_admin_password: "{{ vault_portainer_password }}"
portainer_url: "{{ portainer_protocol }}://{{ ansible_host }}:{{ portainer_port }}"
portainer_api_url: "{{ portainer_url }}/api"

# Stack settings
default_stack_prune: true
default_pull_image: true
```

```yaml
# group_vars/vault.yml (encrypt with ansible-vault)
vault_portainer_password: "your-secure-password"
```

## Step 3: Create Portainer Authentication Module

Write a reusable task file for Portainer authentication:

```yaml
# tasks/portainer_auth.yml
---
# Authenticate with Portainer API and get JWT token
- name: Authenticate with Portainer API
  uri:
    url: "{{ portainer_api_url }}/auth"
    method: POST
    body_format: json
    body:
      Username: "{{ portainer_admin_user }}"
      Password: "{{ portainer_admin_password }}"
    validate_certs: false
    status_code: 200
  register: portainer_auth_response
  no_log: true  # Don't log credentials

- name: Set Portainer JWT token
  set_fact:
    portainer_token: "{{ portainer_auth_response.json.jwt }}"
  no_log: true
```

## Step 4: Manage Portainer Endpoints (Environments)

```yaml
# playbooks/manage_environments.yml
---
- name: Manage Portainer Environments
  hosts: portainer_servers
  gather_facts: false
  
  vars:
    docker_environments:
      - name: "Production Docker"
        url: "tcp://prod-docker.example.com:2376"
        type: 1  # 1=Docker, 2=Swarm, 3=Azure, 6=Kubernetes
        tls: true
      - name: "Staging Docker"
        url: "tcp://staging-docker.example.com:2376"
        type: 1
        tls: false

  tasks:
    - name: Include authentication tasks
      include_tasks: ../tasks/portainer_auth.yml

    - name: Get existing environments
      uri:
        url: "{{ portainer_api_url }}/endpoints"
        method: GET
        headers:
          Authorization: "Bearer {{ portainer_token }}"
        validate_certs: false
      register: existing_environments

    - name: Register Docker environments
      uri:
        url: "{{ portainer_api_url }}/endpoints"
        method: POST
        headers:
          Authorization: "Bearer {{ portainer_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          Name: "{{ item.name }}"
          EndpointCreationType: 1
          URL: "{{ item.url }}"
          TLS: "{{ item.tls | default(false) }}"
        validate_certs: false
        status_code: 200
      loop: "{{ docker_environments }}"
      when: item.name not in (existing_environments.json | map(attribute='Name') | list)
      register: env_creation_results
```

## Step 5: Deploy and Manage Stacks

```yaml
# playbooks/deploy_stacks.yml
---
- name: Deploy Application Stacks via Portainer
  hosts: portainer_servers
  gather_facts: false

  vars:
    stacks:
      - name: monitoring
        compose_file: "../stacks/monitoring/docker-compose.yml"
        env_vars:
          GRAFANA_PASSWORD: "{{ vault_grafana_password }}"
          PROMETHEUS_RETENTION: "15d"
      - name: web-app
        compose_file: "../stacks/webapp/docker-compose.yml"
        env_vars:
          DB_PASSWORD: "{{ vault_db_password }}"
          APP_SECRET: "{{ vault_app_secret }}"

  tasks:
    - name: Include authentication tasks
      include_tasks: ../tasks/portainer_auth.yml

    - name: Get environment ID
      uri:
        url: "{{ portainer_api_url }}/endpoints"
        method: GET
        headers:
          Authorization: "Bearer {{ portainer_token }}"
        validate_certs: false
      register: portainer_environments

    - name: Set environment ID
      set_fact:
        env_id: "{{ portainer_environments.json[0].Id }}"

    - name: Read stack compose file
      slurp:
        src: "{{ item.compose_file }}"
      loop: "{{ stacks }}"
      register: compose_files

    - name: Check if stack exists
      uri:
        url: "{{ portainer_api_url }}/stacks"
        method: GET
        headers:
          Authorization: "Bearer {{ portainer_token }}"
        validate_certs: false
      register: existing_stacks

    - name: Deploy or update stacks
      uri:
        url: "{{ portainer_api_url }}/stacks/create/standalone/string?endpointId={{ env_id }}"
        method: POST
        headers:
          Authorization: "Bearer {{ portainer_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          Name: "{{ item.item.name }}"
          StackFileContent: "{{ item.content | b64decode }}"
          Env: "{{ item.item.env_vars | dict2items | map('combine', {'name': omit, 'value': omit}) | list }}"
        validate_certs: false
        status_code: [200, 201]
      loop: "{{ compose_files.results }}"
      when: item.item.name not in (existing_stacks.json | map(attribute='Name') | list)
```

## Step 6: Manage Users and Teams

```yaml
# playbooks/manage_users.yml
---
- name: Manage Portainer Users and Teams
  hosts: portainer_servers
  gather_facts: false

  vars:
    teams:
      - name: "DevOps"
      - name: "Developers"
    
    users:
      - username: "john.doe"
        password: "{{ vault_user_passwords.john_doe }}"
        role: 2  # 1=admin, 2=standard user
        teams: ["DevOps"]
      - username: "jane.smith"
        password: "{{ vault_user_passwords.jane_smith }}"
        role: 2
        teams: ["Developers"]

  tasks:
    - name: Include authentication tasks
      include_tasks: ../tasks/portainer_auth.yml

    - name: Create teams
      uri:
        url: "{{ portainer_api_url }}/teams"
        method: POST
        headers:
          Authorization: "Bearer {{ portainer_token }}"
        body_format: json
        body:
          Name: "{{ item.name }}"
        validate_certs: false
        status_code: [200, 201, 409]  # 409 = already exists
      loop: "{{ teams }}"

    - name: Create users
      uri:
        url: "{{ portainer_api_url }}/users"
        method: POST
        headers:
          Authorization: "Bearer {{ portainer_token }}"
        body_format: json
        body:
          Username: "{{ item.username }}"
          Password: "{{ item.password }}"
          Role: "{{ item.role }}"
        validate_certs: false
        status_code: [200, 201, 409]
        no_log: true
      loop: "{{ users }}"
```

## Step 7: Run the Playbooks

```bash
# Encrypt sensitive variables
ansible-vault encrypt group_vars/vault.yml

# Run environment setup
ansible-playbook -i inventory/hosts playbooks/manage_environments.yml \
  --ask-vault-pass -v

# Deploy stacks
ansible-playbook -i inventory/hosts playbooks/deploy_stacks.yml \
  --ask-vault-pass -v

# Manage users
ansible-playbook -i inventory/hosts playbooks/manage_users.yml \
  --ask-vault-pass -v

# Run all playbooks
ansible-playbook -i inventory/hosts site.yml --ask-vault-pass
```

## Conclusion

Managing Portainer with Ansible provides a powerful, scalable approach to infrastructure automation. The combination of Ansible's idempotent execution model and Portainer's comprehensive REST API enables you to manage hundreds of Portainer instances consistently. Storing playbooks in Git provides version history, peer review through pull requests, and the foundation for a GitOps workflow where infrastructure changes are automated and auditable.
