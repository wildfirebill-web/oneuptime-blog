# How to Automate DHCP Scope Configuration with Ansible on Windows Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ansible, DHCP, Windows Server, Automation, PowerShell, IPv4, Network Management

Description: Learn how to automate DHCP scope creation and management on Windows Server using Ansible with the win_shell module and PowerShell DHCP cmdlets.

---

Windows Server DHCP can be automated with Ansible using PowerShell's `DhcpServer` module via `win_shell` or `win_powershell` tasks.

## Prerequisites

```yaml
# requirements.yml

collections:
  - ansible.windows
  - community.windows
```

```bash
ansible-galaxy collection install ansible.windows community.windows
```

## Inventory for Windows DHCP Server

```ini
[dhcp_servers]
dhcp-01 ansible_host=10.0.0.10

[dhcp_servers:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_user=Administrator
ansible_password="{{ vault_windows_password }}"
ansible_winrm_server_cert_validation=ignore
```

## Creating a DHCP Scope

```yaml
---
- name: Configure DHCP Scopes on Windows Server
  hosts: dhcp_servers
  gather_facts: false

  vars:
    dhcp_scopes:
      - name: "Servers VLAN10"
        scope_id: "10.10.0.0"
        start: "10.10.0.50"
        end: "10.10.0.200"
        subnet: "255.255.255.0"
        description: "Server network"
        gateway: "10.10.0.1"
        dns: ["10.0.0.1", "10.0.0.2"]
        lease_days: 1

  tasks:
    - name: Create DHCP scopes
      ansible.windows.win_powershell:
        script: |
          $scope = Get-DhcpServerv4Scope -ScopeId "{{ item.scope_id }}" -ErrorAction SilentlyContinue
          if (-not $scope) {
            Add-DhcpServerv4Scope `
              -Name "{{ item.name }}" `
              -StartRange "{{ item.start }}" `
              -EndRange "{{ item.end }}" `
              -SubnetMask "{{ item.subnet }}" `
              -Description "{{ item.description }}" `
              -LeaseDuration ([TimeSpan]::FromDays({{ item.lease_days }})) `
              -State Active
            Write-Output "Created scope {{ item.scope_id }}"
          } else {
            Write-Output "Scope {{ item.scope_id }} already exists"
          }
      loop: "{{ dhcp_scopes }}"

    - name: Set DHCP options (gateway and DNS)
      ansible.windows.win_powershell:
        script: |
          Set-DhcpServerv4OptionValue `
            -ScopeId "{{ item.scope_id }}" `
            -Router "{{ item.gateway }}" `
            -DnsServer {{ item.dns | join(',') }}
      loop: "{{ dhcp_scopes }}"
```

## Verifying Scopes

```yaml
    - name: List all DHCP scopes
      ansible.windows.win_powershell:
        script: Get-DhcpServerv4Scope | Select-Object ScopeId, Name, StartRange, EndRange, State | ConvertTo-Json
      register: scope_list

    - name: Show scopes
      debug:
        var: scope_list.output
```

## Activating or Deactivating a Scope

```yaml
    - name: Deactivate test scope
      ansible.windows.win_powershell:
        script: Set-DhcpServerv4Scope -ScopeId "10.10.0.0" -State Inactive
```

## Key Takeaways

- Use `ansible.windows.win_powershell` with PowerShell DHCP cmdlets (`Add-DhcpServerv4Scope`, `Set-DhcpServerv4OptionValue`) to manage Windows DHCP idempotently.
- Store Windows credentials in Ansible Vault; use WinRM with NTLM or Kerberos transport.
- Check for scope existence before creating to make tasks idempotent (PowerShell's `Get-DhcpServerv4Scope -ErrorAction SilentlyContinue`).
- Loop over a `dhcp_scopes` variable list to manage multiple scopes with a single task.
