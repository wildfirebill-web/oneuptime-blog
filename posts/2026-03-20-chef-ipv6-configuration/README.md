# How to Configure IPv6 with Chef

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Chef, IPv6, Configuration Management, Automation, Cookbook, Recipes

Description: A guide to configuring IPv6 network settings on Linux systems using Chef recipes and cookbooks, including sysctl, interface configuration, and firewall rules.

Chef's Ruby-based DSL enables flexible IPv6 configuration management across server fleets. This guide covers creating Chef recipes and cookbooks for IPv6 network configuration.

## Cookbook Structure

```text
cookbooks/ipv6_config/
├── recipes/
│   ├── default.rb       # Main recipe
│   ├── sysctl.rb        # Kernel parameters
│   ├── firewall.rb      # ip6tables rules
│   └── interfaces.rb    # Network config
├── attributes/
│   └── default.rb       # Default attribute values
├── templates/
│   └── sysctl.conf.erb  # sysctl template
└── metadata.rb
```

## Default Attributes

```ruby
# attributes/default.rb

default['ipv6_config']['disable_ipv6'] = false
default['ipv6_config']['enable_forwarding'] = false
default['ipv6_config']['accept_ra'] = 1
default['ipv6_config']['privacy_extensions'] = true
default['ipv6_config']['use_tempaddr'] = 2    # 2 = prefer temporary addresses
```

## Main Recipe

```ruby
# recipes/default.rb

include_recipe 'ipv6_config::sysctl'

if node['ipv6_config']['privacy_extensions']
  include_recipe 'ipv6_config::privacy'
end

if node['ipv6_config']['enable_forwarding']
  include_recipe 'ipv6_config::routing'
end
```

## sysctl Recipe

```ruby
# recipes/sysctl.rb

# Configure IPv6 kernel parameters

sysctl_param 'net.ipv6.conf.all.disable_ipv6' do
  value node['ipv6_config']['disable_ipv6'] ? 1 : 0
end

sysctl_param 'net.ipv6.conf.default.disable_ipv6' do
  value node['ipv6_config']['disable_ipv6'] ? 1 : 0
end

sysctl_param 'net.ipv6.conf.all.accept_ra' do
  value node['ipv6_config']['accept_ra']
end

sysctl_param 'net.ipv6.conf.all.forwarding' do
  value node['ipv6_config']['enable_forwarding'] ? 1 : 0
end

# If using sysctl resource directly:
template '/etc/sysctl.d/60-ipv6.conf' do
  source 'sysctl.conf.erb'
  mode '0644'
  variables(
    disable_ipv6: node['ipv6_config']['disable_ipv6'],
    forwarding: node['ipv6_config']['enable_forwarding'],
    accept_ra: node['ipv6_config']['accept_ra'],
    use_tempaddr: node['ipv6_config']['use_tempaddr']
  )
  notifies :run, 'execute[reload_sysctl]', :immediately
end

execute 'reload_sysctl' do
  command 'sysctl --system'
  action :nothing
end
```

## sysctl Template

```erb
<%# templates/sysctl.conf.erb %>
# IPv6 Configuration managed by Chef - do not edit manually
# Managed by: <%= node['chef_environment'] %>

net.ipv6.conf.all.disable_ipv6 = <%= @disable_ipv6 ? 1 : 0 %>
net.ipv6.conf.default.disable_ipv6 = <%= @disable_ipv6 ? 1 : 0 %>
net.ipv6.conf.all.forwarding = <%= @forwarding ? 1 : 0 %>
net.ipv6.conf.default.forwarding = <%= @forwarding ? 1 : 0 %>
net.ipv6.conf.all.accept_ra = <%= @accept_ra %>
net.ipv6.conf.all.use_tempaddr = <%= @use_tempaddr %>
```

## Firewall Recipe

```ruby
# recipes/firewall.rb

# Using the chef community 'firewall' cookbook for ip6tables
firewall 'default' do
  ipv6_enabled true
  action :install
end

firewall_rule 'allow_ipv6_icmp' do
  protocol :icmpv6
  command :allow
  source '::/0'
end

firewall_rule 'allow_ipv6_ssh' do
  port 22
  protocol :tcp
  source '::/0'
  command :allow
end

firewall_rule 'allow_ipv6_http' do
  port [80, 443]
  protocol :tcp
  source '::/0'
  command :allow
end
```

## Interface Configuration

```ruby
# recipes/interfaces.rb

# Configure Netplan (Ubuntu 18.04+)
template '/etc/netplan/99-ipv6-chef.yaml' do
  source 'netplan-ipv6.yaml.erb'
  mode '0600'
  variables(
    interface: node['ipv6_config']['interface'] || 'eth0',
    static_address: node['ipv6_config']['static_address'],
    gateway: node['ipv6_config']['gateway']
  )
  notifies :run, 'execute[netplan_apply]', :delayed
end

execute 'netplan_apply' do
  command 'netplan apply'
  action :nothing
end
```

## Chef Roles for IPv6

```json
// roles/ipv6-web-server.json
{
  "name": "ipv6-web-server",
  "description": "IPv6-enabled web server",
  "run_list": [
    "recipe[ipv6_config::default]",
    "recipe[ipv6_config::firewall]"
  ],
  "default_attributes": {
    "ipv6_config": {
      "disable_ipv6": false,
      "enable_forwarding": false,
      "privacy_extensions": true
    }
  }
}
```

## Applying the Cookbook

```bash
# Upload cookbook
knife cookbook upload ipv6_config

# Assign role to node
knife node run_list add webserver01 'role[ipv6-web-server]'

# Run chef-client on node
ssh webserver01 'sudo chef-client'

# Verify result
ssh webserver01 'ip -6 addr show; sysctl net.ipv6.conf.all.forwarding'
```

Chef cookbooks for IPv6 configuration provide reusable, testable infrastructure code that can be applied consistently across development, staging, and production environments with role-based customization.
