# How to Validate IPv6 Configuration with Ansible Assertions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, IPv6, Validation, Assertions, Testing, Compliance

Description: A guide to using Ansible's assert module to validate IPv6 configuration compliance on Linux hosts and network devices.

Ansible's `assert` module transforms configuration checks into auditable validation tasks. Using assertions for IPv6 configuration ensures every host meets your networking standards before services go live.

## Basic IPv6 Address Assertions

```yaml
# validate-ipv6-address.yml - Assert IPv6 address is correctly configured

---
- name: Validate IPv6 address configuration
  hosts: all
  become: true

  vars:
    # Expected IPv6 address for this host (from inventory or group_vars)
    expected_ipv6: "{{ hostvars[inventory_hostname]['ipv6_address'] | default('') }}"

  tasks:
    - name: Gather IPv6 addresses on all interfaces
      ansible.builtin.command:
        cmd: ip -6 addr show scope global
      register: ipv6_addrs
      changed_when: false

    - name: Assert at least one global IPv6 address exists
      ansible.builtin.assert:
        that:
          - "'inet6' in ipv6_addrs.stdout"
        fail_msg: "{{ inventory_hostname }} has no global IPv6 address"
        success_msg: "{{ inventory_hostname }} has a global IPv6 address"

    - name: Assert the expected IPv6 address is present
      ansible.builtin.assert:
        that:
          - "expected_ipv6 in ipv6_addrs.stdout"
        fail_msg: "Expected IPv6 {{ expected_ipv6 }} not found on {{ inventory_hostname }}"
      when: expected_ipv6 != ''
```

## sysctl Settings Compliance Assertions

```yaml
# validate-ipv6-sysctl.yml - Assert sysctl settings are correctly configured
---
- name: Validate IPv6 sysctl compliance
  hosts: all
  become: true

  vars:
    required_sysctl:
      - { param: "net.ipv6.conf.all.forwarding", value: "0" }
      - { param: "net.ipv6.conf.all.accept_ra", value: "1" }
      - { param: "net.ipv6.conf.all.disable_ipv6", value: "0" }

  tasks:
    - name: Read each sysctl parameter
      ansible.builtin.command:
        cmd: "sysctl -n {{ item.param }}"
      register: sysctl_results
      loop: "{{ required_sysctl }}"
      changed_when: false

    - name: Assert each sysctl parameter has the expected value
      ansible.builtin.assert:
        that:
          - item.stdout == item.item.value
        fail_msg: >
          sysctl {{ item.item.param }} = {{ item.stdout }},
          expected {{ item.item.value }} on {{ inventory_hostname }}
        success_msg: "sysctl {{ item.item.param }} is correctly set to {{ item.item.value }}"
      loop: "{{ sysctl_results.results }}"
```

## Service Listening Assertions

```yaml
# validate-ipv6-services.yml - Assert services listen on IPv6
---
  tasks:
    - name: Get listening ports
      ansible.builtin.command:
        cmd: ss -6 -tlnp
      register: listening_ports
      changed_when: false

    - name: Assert nginx listens on IPv6 port 80
      ansible.builtin.assert:
        that:
          - "':80' in listening_ports.stdout"
        fail_msg: "nginx is NOT listening on IPv6 port 80"

    - name: Assert SSH listens on IPv6 port 22
      ansible.builtin.assert:
        that:
          - "':22' in listening_ports.stdout"
        fail_msg: "SSH is NOT listening on IPv6 port 22"
```

## Connectivity Assertions

```yaml
# validate-ipv6-connectivity.yml - Assert IPv6 connectivity is working
---
  tasks:
    - name: Test IPv6 connectivity to DNS resolver
      ansible.builtin.command:
        cmd: ping6 -c 3 -W 5 2001:4860:4860::8888
      register: ping_result
      changed_when: false
      failed_when: false

    - name: Assert IPv6 external connectivity
      ansible.builtin.assert:
        that:
          - ping_result.rc == 0
        fail_msg: "Cannot reach IPv6 internet from {{ inventory_hostname }}"
        success_msg: "IPv6 external connectivity verified on {{ inventory_hostname }}"

    - name: Test IPv6 DNS resolution
      ansible.builtin.command:
        cmd: dig AAAA google.com +short
      register: dns_result
      changed_when: false

    - name: Assert IPv6 DNS returns AAAA records
      ansible.builtin.assert:
        that:
          - dns_result.stdout | length > 0
          - "':' in dns_result.stdout"
        fail_msg: "IPv6 DNS is not returning AAAA records on {{ inventory_hostname }}"
```

## Full Compliance Playbook

```yaml
# full-ipv6-audit.yml - Run all IPv6 compliance checks
---
- name: Full IPv6 Configuration Audit
  hosts: all
  become: true

  tasks:
    - name: Run IPv6 address validation
      ansible.builtin.include_tasks: validate-ipv6-address.yml

    - name: Run sysctl compliance checks
      ansible.builtin.include_tasks: validate-ipv6-sysctl.yml

    - name: Run service listening checks
      ansible.builtin.include_tasks: validate-ipv6-services.yml

    - name: Run connectivity checks
      ansible.builtin.include_tasks: validate-ipv6-connectivity.yml
```

## Run the Audit

```bash
# Run the full audit without making changes
ansible-playbook full-ipv6-audit.yml -i inventory.ini

# Generate a report
ansible-playbook full-ipv6-audit.yml -i inventory.ini \
  --extra-vars "ansible_check_mode=true" 2>&1 | tee ipv6-audit-report.txt
```

Ansible assertions provide a simple, declarative way to enforce IPv6 configuration standards across your entire infrastructure and generate compliance reports without modifying any system state.
