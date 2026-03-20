# How to Automate IPv6 Firewall Rules with Chef

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Chef, IPv6, Firewall, ip6tables, Automation, Security

Description: A guide to automating IPv6 firewall rule management with Chef using the firewall cookbook and custom resources for consistent ip6tables deployment.

Chef provides the `firewall` community cookbook for managing ip6tables rules declaratively. This guide covers implementing a comprehensive IPv6 firewall policy using Chef resources, with support for different server roles.

## Setup: Installing the firewall Cookbook

```ruby
# Berksfile
source 'https://supermarket.chef.io'
cookbook 'firewall', '~> 2.7'
```

```bash
berks install && berks upload
```

## Base IPv6 Firewall Recipe

```ruby
# cookbooks/ipv6_firewall/recipes/base.rb

# Include the firewall cookbook
include_recipe 'firewall'

# Enable ip6tables
firewall 'default' do
  ipv6_enabled true
  action [:install, :save]
end

# Allow loopback interface
firewall_rule 'ipv6_loopback' do
  interface 'lo'
  command :allow
  position 1
end

# Allow established and related connections
firewall_rule 'ipv6_established' do
  stateful [:established, :related]
  command :allow
  position 2
end

# CRITICAL: Allow ICMPv6 (breaks IPv6 if blocked)
# Packet Too Big, NDP, Router Discovery all require ICMPv6
firewall_rule 'ipv6_icmpv6_all' do
  protocol :icmpv6
  source '::/0'
  command :allow
  position 3
end

# Drop all other INPUT traffic
firewall_rule 'ipv6_drop_all' do
  command :deny
  direction :in
  position 9999
end
```

## Role-Based IPv6 Firewall Rules

```ruby
# cookbooks/ipv6_firewall/recipes/web_server.rb

include_recipe 'ipv6_firewall::base'

# Allow HTTP and HTTPS from any IPv6
firewall_rule 'ipv6_http' do
  port 80
  protocol :tcp
  source '::/0'
  command :allow
  position 100
end

firewall_rule 'ipv6_https' do
  port 443
  protocol :tcp
  source '::/0'
  command :allow
  position 101
end
```

```ruby
# cookbooks/ipv6_firewall/recipes/database.rb

include_recipe 'ipv6_firewall::base'

# Allow PostgreSQL only from application server subnet
firewall_rule 'ipv6_postgres' do
  port 5432
  protocol :tcp
  source '2001:db8:app::/48'    # App server IPv6 subnet
  command :allow
  position 100
end

firewall_rule 'ipv6_redis' do
  port 6379
  protocol :tcp
  source '2001:db8:app::/48'
  command :allow
  position 101
end
```

## Custom Resource for IPv6 Rules

```ruby
# cookbooks/ipv6_firewall/resources/rule.rb

resource_name :ipv6_rule
provides :ipv6_rule

property :port, [Integer, Array]
property :source, String, default: '::/0'
property :protocol, Symbol, default: :tcp
property :command, Symbol, default: :allow
property :position, Integer, default: 100

action :create do
  firewall_rule new_resource.name do
    port     new_resource.port
    source   new_resource.source
    protocol new_resource.protocol
    command  new_resource.command
    position new_resource.position
  end
end
```

## Attribute-Driven Firewall Configuration

```ruby
# attributes/default.rb

# Default IPv6 firewall rules
default['ipv6_firewall']['rules'] = {
  'ssh' => {
    'port' => 22,
    'source' => '::/0',
    'position' => 50
  }
}
```

```ruby
# recipes/attribute_driven.rb

include_recipe 'ipv6_firewall::base'

# Apply rules from attributes
node['ipv6_firewall']['rules'].each do |name, config|
  firewall_rule "ipv6_#{name}" do
    port      config['port']
    source    config.fetch('source', '::/0')
    protocol  config.fetch('protocol', :tcp).to_sym
    command   config.fetch('command', :allow).to_sym
    position  config.fetch('position', 100)
  end
end
```

## ip6tables Direct Recipe (without firewall cookbook)

```ruby
# recipes/ip6tables_direct.rb

# Manage ip6tables rules without the firewall cookbook
package 'iptables-persistent' do
  action :install
end

file '/etc/ip6tables.rules' do
  content <<~RULES
    # Managed by Chef
    *filter
    :INPUT DROP [0:0]
    :FORWARD DROP [0:0]
    :OUTPUT ACCEPT [0:0]
    -A INPUT -i lo -j ACCEPT
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    -A INPUT -p ipv6-icmp -j ACCEPT
    -A INPUT -p tcp --dport 22 -j ACCEPT
    #{node['ipv6_firewall']['rules'].map { |n, c| "-A INPUT -p tcp --dport #{c['port']} -j ACCEPT" }.join("\n")}
    COMMIT
  RULES
  notifies :run, 'execute[reload_ip6tables]', :immediately
end

execute 'reload_ip6tables' do
  command 'ip6tables-restore < /etc/ip6tables.rules'
  action :nothing
end
```

## Testing IPv6 Firewall with InSpec

```ruby
# test/integration/default/firewall_test.rb

describe 'IPv6 Firewall' do
  # Test that ip6tables is running
  describe service('netfilter-persistent') do
    it { should be_running }
    it { should be_enabled }
  end

  # Test ICMPv6 is allowed
  describe command('ip6tables -L INPUT -n') do
    its('stdout') { should match /ACCEPT.*ipv6-icmp/ }
  end

  # Test HTTP is allowed
  describe command('ip6tables -L INPUT -n') do
    its('stdout') { should match /ACCEPT.*tcp.*dpt:443/ }
  end

  # Test default policy is DROP
  describe command('ip6tables -L INPUT -n') do
    its('stdout') { should match /Chain INPUT \(policy DROP\)/ }
  end
end
```

Chef's firewall cookbook and resource model makes IPv6 firewall management declarative and testable, ensuring consistent security policies across all managed nodes with role-based rule customization through attributes and recipes.
