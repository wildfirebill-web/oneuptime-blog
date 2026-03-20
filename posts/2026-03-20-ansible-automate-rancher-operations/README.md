# How to Use Ansible to Automate Rancher Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Ansible, Automation, Infrastructure as Code, DevOps, Kubernetes

Description: Automate Rancher cluster management, user provisioning, and application deployments using Ansible playbooks with the Rancher API.

## Introduction

Ansible is an agentless automation tool that works well for orchestrating Rancher operations via its REST API. Unlike Terraform, Ansible is better suited for procedural tasks: running health checks, rotating credentials, deploying applications in sequence, and performing pre/post migration steps.

## Prerequisites

- Ansible 2.12+
- `kubectl` and a valid kubeconfig
- Rancher API token

## Step 1: Install Required Ansible Collections

```bash
# Install the Kubernetes and community.general collections
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install community.general
```

## Step 2: Configure Inventory and Variables

```yaml
# inventory/group_vars/all.yml
rancher_url: "https://rancher.example.com"
rancher_token: "{{ lookup('env', 'RANCHER_TOKEN') }}"

clusters:
  - name: production
    id: c-m-abc123
  - name: staging
    id: c-m-def456
```

## Step 3: Create a Namespace Provisioning Playbook

```yaml
# playbooks/provision-namespace.yml
---
- name: Provision application namespaces in Rancher
  hosts: localhost
  gather_facts: false

  vars:
    app_name: "myapp"
    environment: "production"

  tasks:
    - name: Create namespace via Rancher API
      ansible.builtin.uri:
        url: "{{ rancher_url }}/v3/cluster/{{ cluster_id }}/namespace"
        method: POST
        headers:
          Authorization: "Bearer {{ rancher_token }}"
          Content-Type: "application/json"
        body_format: json
        body:
          name: "{{ app_name }}-{{ environment }}"
          projectId: "{{ project_id }}"
          labels:
            environment: "{{ environment }}"
            app: "{{ app_name }}"
        status_code: [200, 201]
      register: namespace_result

    - name: Display namespace creation result
      debug:
        msg: "Created namespace: {{ namespace_result.json.name }}"
```

## Step 4: Deploy an Application with Ansible

```yaml
# playbooks/deploy-app.yml
---
- name: Deploy application to Rancher cluster
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Apply Kubernetes manifests using kubectl
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        state: present
        definition: "{{ lookup('template', 'templates/deployment.yaml.j2') }}"

    - name: Wait for deployment to be ready
      kubernetes.core.k8s_info:
        kubeconfig: "{{ kubeconfig_path }}"
        kind: Deployment
        name: "{{ app_name }}"
        namespace: "{{ app_namespace }}"
        wait: true
        wait_condition:
          type: Available
          status: "True"
        wait_timeout: 300

    - name: Verify pods are running
      kubernetes.core.k8s_info:
        kubeconfig: "{{ kubeconfig_path }}"
        kind: Pod
        namespace: "{{ app_namespace }}"
        label_selectors:
          - app={{ app_name }}
      register: pod_info

    - name: Print pod status
      debug:
        msg: "Pod {{ item.metadata.name }} is {{ item.status.phase }}"
      loop: "{{ pod_info.resources }}"
```

## Step 5: Run the Playbook

```bash
# Set the Rancher token as an environment variable
export RANCHER_TOKEN="token-xxxxx:yyyyy"

# Run the provisioning playbook
ansible-playbook playbooks/provision-namespace.yml \
  -e "cluster_id=c-m-abc123" \
  -e "project_id=c-m-abc123:p-12345"

# Deploy the application
ansible-playbook playbooks/deploy-app.yml \
  -e "app_name=myapp" \
  -e "app_namespace=myapp-production"
```

## Conclusion

Ansible complements Terraform for Rancher automation by handling procedural workflows that don't fit the declarative model. Use Terraform for cluster provisioning and Ansible for post-provisioning configuration, health verification, and ongoing operational tasks like rolling updates and credential rotation.
