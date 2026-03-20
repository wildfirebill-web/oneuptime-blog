# How to Configure Organization-Wide IPv6 Privacy Policies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Privacy, Policy, Enterprise, Linux, Ansible

Description: Implement organization-wide IPv6 privacy policies by deploying consistent address generation settings across all managed systems using configuration management tools.

## Introduction

Ensuring consistent IPv6 privacy settings across an entire organization requires a policy-driven approach rather than ad-hoc per-system configuration. This guide covers defining a privacy policy, enforcing it via configuration management, and auditing compliance at scale.

## Defining Your Privacy Policy

Before deploying, define the policy in writing:

```
# IPv6 Privacy Policy v1.0
# All managed Linux systems must:
# 1. Use stable-privacy (RFC 7217) for address generation (addr_gen_mode=2)
# 2. Generate temporary addresses (use_tempaddr=2)
# 3. Temporary preferred lifetime: 14400s (4 hours)
# 4. Temporary valid lifetime: 86400s (24 hours)
# 5. No EUI-64 addresses permitted on any global-scope interface
```

## Enforcing via Ansible

Create an Ansible role that enforces the IPv6 privacy policy:

```yaml
# roles/ipv6_privacy/tasks/main.yml
---
- name: Deploy IPv6 privacy sysctl configuration
  ansible.builtin.copy:
    dest: /etc/sysctl.d/60-ipv6-privacy.conf
    content: |
      # Organization IPv6 Privacy Policy - managed by Ansible
      # Do not edit manually
      net.ipv6.conf.default.addr_gen_mode = 2
      net.ipv6.conf.all.addr_gen_mode = 2
      net.ipv6.conf.default.use_tempaddr = 2
      net.ipv6.conf.all.use_tempaddr = 2
      net.ipv6.conf.default.temp_prefered_lft = 14400
      net.ipv6.conf.all.temp_prefered_lft = 14400
      net.ipv6.conf.default.temp_valid_lft = 86400
      net.ipv6.conf.all.temp_valid_lft = 86400
    owner: root
    group: root
    mode: "0644"
  notify: Apply sysctl

- name: Deploy NetworkManager global config for stable-privacy
  ansible.builtin.copy:
    dest: /etc/NetworkManager/conf.d/60-ipv6-privacy.conf
    content: |
      [connection]
      ipv6.addr-gen-mode=stable-privacy
    owner: root
    group: root
    mode: "0644"
  when: ansible_facts.services['NetworkManager.service'] is defined
  notify: Reload NetworkManager
```

```yaml
# roles/ipv6_privacy/handlers/main.yml
---
- name: Apply sysctl
  ansible.builtin.command: sysctl --system
  listen: Apply sysctl

- name: Reload NetworkManager
  ansible.builtin.systemd:
    name: NetworkManager
    state: reloaded
  listen: Reload NetworkManager
```

## Puppet Module

```puppet
# modules/ipv6_privacy/manifests/init.pp
class ipv6_privacy {
  # Deploy sysctl settings for IPv6 privacy
  file { '/etc/sysctl.d/60-ipv6-privacy.conf':
    ensure  => file,
    owner   => 'root',
    group   => 'root',
    mode    => '0644',
    content => @("END"),
      # Managed by Puppet - IPv6 Privacy Policy
      net.ipv6.conf.default.addr_gen_mode = 2
      net.ipv6.conf.all.addr_gen_mode = 2
      net.ipv6.conf.default.use_tempaddr = 2
      net.ipv6.conf.all.use_tempaddr = 2
      END
    notify  => Exec['sysctl-reload'],
  }

  exec { 'sysctl-reload':
    command     => '/sbin/sysctl --system',
    refreshonly => true,
  }
}
```

## Chef Recipe

```ruby
# cookbooks/ipv6_privacy/recipes/default.rb

# Write the sysctl configuration file
file '/etc/sysctl.d/60-ipv6-privacy.conf' do
  content <<~CONF
    # Chef-managed: IPv6 Privacy Policy
    net.ipv6.conf.default.addr_gen_mode = 2
    net.ipv6.conf.all.addr_gen_mode = 2
    net.ipv6.conf.default.use_tempaddr = 2
    net.ipv6.conf.all.use_tempaddr = 2
  CONF
  owner 'root'
  group 'root'
  mode '0644'
  notifies :run, 'execute[apply-sysctl]', :immediately
end

execute 'apply-sysctl' do
  command 'sysctl --system'
  action :nothing
end
```

## Policy Compliance Reporting

Generate a compliance report across all managed hosts:

```bash
#!/bin/bash
# report_ipv6_compliance.sh
# SSH to all hosts and check compliance

HOSTS_FILE="$1"
REPORT="ipv6_compliance_$(date +%Y%m%d).csv"

echo "hostname,addr_gen_mode_all,use_tempaddr_all,compliant" > "$REPORT"

while IFS= read -r host; do
    result=$(ssh -o ConnectTimeout=5 "$host" \
        "echo \$(sysctl -n net.ipv6.conf.all.addr_gen_mode),\$(sysctl -n net.ipv6.conf.all.use_tempaddr)" 2>/dev/null)

    if [ -z "$result" ]; then
        echo "$host,ERROR,ERROR,UNKNOWN" >> "$REPORT"
        continue
    fi

    gen_mode=$(echo "$result" | cut -d, -f1)
    temp_addr=$(echo "$result" | cut -d, -f2)

    if [ "$gen_mode" = "2" ] && [ "$temp_addr" = "2" ]; then
        compliant="YES"
    else
        compliant="NO"
    fi

    echo "$host,$gen_mode,$temp_addr,$compliant" >> "$REPORT"
done < "$HOSTS_FILE"

echo "Report saved to $REPORT"
```

## Conclusion

Organization-wide IPv6 privacy policies are most effective when deployed and enforced through configuration management systems like Ansible, Puppet, or Chef. Define the policy explicitly in documentation, implement it as code, and schedule regular compliance checks. This ensures that all systems — from newly provisioned VMs to long-running bare-metal servers — maintain consistent, auditable IPv6 privacy settings.
