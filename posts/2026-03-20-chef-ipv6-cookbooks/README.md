# How to Deploy IPv6 Configuration with Chef Cookbooks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Chef, IPv6, Cookbooks, Supermarket, Configuration Management, Automation

Description: A guide to using Chef Supermarket cookbooks for IPv6 configuration, including the sysctl, firewall, and network cookbooks for standardized IPv6 deployment.

The Chef Supermarket provides community cookbooks that simplify IPv6 deployment. Leveraging existing cookbooks reduces implementation time and provides tested, community-maintained code for common IPv6 configuration tasks.

## Key Chef Supermarket Cookbooks for IPv6

| Cookbook | Purpose |
|---|---|
| `sysctl` | Manage sysctl parameters persistently |
| `firewall` | Cross-platform firewall management (iptables/ip6tables) |
| `network_interfaces_v2` | Network interface management |
| `os-hardening` | Security hardening including IPv6 settings |

## Installing Cookbooks with Berkshelf

```ruby
# Berksfile

source 'https://supermarket.chef.io'

cookbook 'sysctl', '~> 1.1'
cookbook 'firewall', '~> 2.7'
cookbook 'network_interfaces_v2', '~> 2.2'
```

```bash
# Install all cookbooks
berks install
berks upload
```

## IPv6 sysctl with the sysctl Cookbook

```ruby
# recipes/ipv6_sysctl.rb

include_recipe 'sysctl::default'

# IPv6 parameters
sysctl_param 'net.ipv6.conf.all.disable_ipv6' do
  value 0
end

sysctl_param 'net.ipv6.conf.default.disable_ipv6' do
  value 0
end

sysctl_param 'net.ipv6.conf.all.accept_ra' do
  value 1
end

sysctl_param 'net.ipv6.conf.all.forwarding' do
  value node['ipv6']['forwarding'] ? 1 : 0
end

sysctl_param 'net.ipv6.conf.all.use_tempaddr' do
  value node['ipv6']['privacy']['use_tempaddr']
end
```

## IPv6 Firewall with the firewall Cookbook

The `firewall` cookbook supports both iptables and ip6tables:

```ruby
# recipes/ipv6_firewall.rb

include_recipe 'firewall'

# Enable ip6tables
firewall 'default' do
  ipv6_enabled true
  action [:install, :save]
end

# Allow loopback (both IPv4 and IPv6)
firewall_rule 'loopback' do
  interface 'lo'
  command :allow
end

# Allow established sessions
firewall_rule 'established' do
  stateful [:established, :related]
  command :allow
end

# Allow ICMPv6 (required for IPv6 operations)
firewall_rule 'icmpv6' do
  protocol :icmpv6
  source '::/0'
  command :allow
end

# Allow SSH over IPv6
firewall_rule 'ssh_ipv6' do
  port 22
  protocol :tcp
  source '::/0'
  command :allow
end

# Allow HTTP and HTTPS over IPv6
firewall_rule 'web_ipv6' do
  port [80, 443]
  protocol :tcp
  source '::/0'
  command :allow
end

# Drop everything else
firewall_rule 'drop_all' do
  command :deny
end
```

## Network Interface Configuration

```ruby
# recipes/ipv6_interface.rb

include_recipe 'network_interfaces_v2'

# Configure eth0 with static IPv6
network_interface 'eth0' do
  target_device 'eth0'
  bootproto 'static'
  ipaddress '10.0.0.10'
  netmask '255.255.255.0'
  # Additional IPv6 via post-up commands
  post_up [
    "ip -6 addr add 2001:db8::10/64 dev eth0",
    "ip -6 route add default via 2001:db8::1",
  ]
  pre_down ["ip -6 addr del 2001:db8::10/64 dev eth0"]
end
```

## Wrapper Cookbook Pattern

Create a wrapper cookbook that customizes community cookbooks:

```ruby
# my_company_ipv6/recipes/default.rb

# Override default attributes for sysctl cookbook
node.default['sysctl']['conf_dir'] = '/etc/sysctl.d'

# Override firewall defaults
node.default['firewall']['ipv6_enabled'] = true
node.default['firewall']['log_denied_packets'] = 'unicast'

# Include community cookbooks
include_recipe 'sysctl::default'
include_recipe 'firewall::default'

# Apply IPv6 sysctl settings
sysctl_param 'net.ipv6.conf.all.disable_ipv6' do
  value 0
end

# Apply IPv6 firewall settings
firewall 'default' do
  ipv6_enabled true
  action [:install, :save]
end
```

## Chef Testing for IPv6 Cookbooks

```ruby
# test/integration/default/ipv6_test.rb

describe 'IPv6 Configuration' do
  # Test sysctl settings
  describe kernel_parameter('net.ipv6.conf.all.disable_ipv6') do
    its('value') { should eq 0 }
  end

  describe kernel_parameter('net.ipv6.conf.all.accept_ra') do
    its('value') { should eq 1 }
  end

  # Test IPv6 firewall
  describe command('ip6tables -L INPUT -n') do
    its('stdout') { should match /ACCEPT.*ipv6-icmp/ }
    its('stdout') { should match /ACCEPT.*tcp.*dpt:22/ }
  end

  # Test IPv6 connectivity
  describe command('ip -6 addr show | grep "inet6"') do
    its('stdout') { should_not be_empty }
  end
end
```

```bash
# Run tests with Kitchen
kitchen test

# Or run specific test suite
kitchen converge && kitchen verify
```

Using Chef Supermarket cookbooks for IPv6 deployment provides tested, maintained implementations that reduce development time while ensuring consistent IPv6 configuration across your infrastructure.
