# How to Manage ClickHouse with Chef

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Chef, Configuration Management, DevOps, Cookbook

Description: Learn how to install and manage ClickHouse using a Chef cookbook with recipes, attributes, and templates for reproducible server configuration.

---

## Why Chef for ClickHouse

Chef cookbooks provide a Ruby DSL for managing infrastructure. For ClickHouse, you can write a cookbook that handles repository setup, package installation, config file rendering, and service management in a composable and testable way.

## Cookbook Structure

```text
cookbooks/clickhouse/
  attributes/
    default.rb
  recipes/
    default.rb
    install.rb
    configure.rb
    service.rb
  templates/
    default/
      config.xml.erb
```

## Attributes (attributes/default.rb)

```ruby
default['clickhouse']['version']         = 'latest'
default['clickhouse']['listen_host']     = '0.0.0.0'
default['clickhouse']['max_connections'] = 4096
default['clickhouse']['log_level']       = 'warning'
default['clickhouse']['data_dir']        = '/var/lib/clickhouse'
```

## Install Recipe (recipes/install.rb)

```ruby
apt_repository 'clickhouse' do
  uri          'https://packages.clickhouse.com/deb'
  distribution 'lts'
  components   ['main']
  key          'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key'
  action       :add
end

%w[clickhouse-server clickhouse-client].each do |pkg|
  package pkg do
    action :install
  end
end
```

## Configure Recipe (recipes/configure.rb)

```ruby
template '/etc/clickhouse-server/config.xml' do
  source 'config.xml.erb'
  owner  'clickhouse'
  group  'clickhouse'
  mode   '0640'
  variables(
    listen_host:     node['clickhouse']['listen_host'],
    max_connections: node['clickhouse']['max_connections'],
    log_level:       node['clickhouse']['log_level'],
    data_dir:        node['clickhouse']['data_dir']
  )
  notifies :restart, 'service[clickhouse-server]', :delayed
end
```

## Service Recipe (recipes/service.rb)

```ruby
service 'clickhouse-server' do
  action  [:enable, :start]
  supports restart: true, status: true
end
```

## Default Recipe (recipes/default.rb)

```ruby
include_recipe 'clickhouse::install'
include_recipe 'clickhouse::configure'
include_recipe 'clickhouse::service'
```

## Running the Cookbook

Add it to a node's run list:

```bash
knife node run_list add ch01.example.com 'recipe[clickhouse]'
chef-client --node-name ch01.example.com
```

Or for local testing with Test Kitchen:

```bash
kitchen converge
kitchen verify
```

## Summary

A Chef clickhouse cookbook separates concerns into `install`, `configure`, and `service` recipes. Use attributes for environment-specific values, templates for config file generation, and `notifies :restart` to reload ClickHouse only when configuration changes. Test Kitchen makes it easy to validate the cookbook against multiple platforms before deploying to production.
