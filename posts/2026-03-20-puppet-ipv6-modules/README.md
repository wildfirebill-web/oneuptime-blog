# How to Deploy IPv6 Configuration with Puppet Modules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Puppet, IPv6, Module, Forge, Configuration Management, Network

Description: A guide to using Puppet Forge modules for IPv6 network configuration, including the network, firewall, and sysctl modules for deploying IPv6 at scale.

The Puppet Forge provides community-maintained modules that simplify IPv6 configuration. Rather than writing all configuration from scratch, leverage existing modules for network interface management, ip6tables firewall rules, and sysctl parameters.

## Key Puppet Forge Modules for IPv6

| Module | Author | Purpose |
|---|---|---|
| `puppetlabs/firewall` | Puppet Labs | ip6tables management |
| `herculesteam/augeasproviders_sysctl` | Hercules Team | sysctl management |
| `puppet/network` | Various | Network interface config |
| `saz/sysctl` | saz | Alternative sysctl module |

## Installing Modules

```bash
# Install required modules

puppet module install puppetlabs-firewall
puppet module install herculesteam-augeasproviders_sysctl
puppet module install puppet-network

# List installed modules
puppet module list
```

## IPv6 sysctl with augeasproviders_sysctl

```puppet
# Manage IPv6 kernel parameters declaratively
class profile::ipv6::sysctl {
  include sysctl::base

  sysctl { 'net.ipv6.conf.all.disable_ipv6':
    value => '0',
  }

  sysctl { 'net.ipv6.conf.default.disable_ipv6':
    value => '0',
  }

  sysctl { 'net.ipv6.conf.all.accept_ra':
    value => '1',
  }

  sysctl { 'net.ipv6.conf.all.forwarding':
    value => '0',
  }

  sysctl { 'net.ipv6.conf.all.use_tempaddr':
    value => '1',
  }
}
```

## IPv6 Firewall with puppetlabs/firewall

The puppetlabs/firewall module supports both iptables and ip6tables:

```puppet
class profile::ipv6::firewall {
  # Ensure ip6tables service is running
  service { 'ip6tables':
    ensure => running,
    enable => true,
  }

  # Purge unmanaged ip6tables rules
  resources { 'firewall':
    purge => true,
  }

  # Rules with provider => ip6tables target IPv6
  Firewall {
    before  => Class['profile::ipv6::firewall::post'],
    require => Class['profile::ipv6::firewall::pre'],
  }

  class { ['profile::ipv6::firewall::pre', 'profile::ipv6::firewall::post']: }
}

class profile::ipv6::firewall::pre {
  firewall { '000 ip6 accept loopback':
    provider => ip6tables,
    chain    => 'INPUT',
    iniface  => 'lo',
    action   => 'accept',
  }

  firewall { '001 ip6 accept established':
    provider => ip6tables,
    chain    => 'INPUT',
    state    => ['ESTABLISHED', 'RELATED'],
    action   => 'accept',
  }

  firewall { '002 ip6 accept icmpv6':
    provider => ip6tables,
    chain    => 'INPUT',
    proto    => 'ipv6-icmp',
    action   => 'accept',
  }

  firewall { '010 ip6 accept ssh':
    provider => ip6tables,
    chain    => 'INPUT',
    dport    => 22,
    proto    => 'tcp',
    action   => 'accept',
  }

  firewall { '020 ip6 accept http https':
    provider => ip6tables,
    chain    => 'INPUT',
    dport    => [80, 443],
    proto    => 'tcp',
    action   => 'accept',
  }
}

class profile::ipv6::firewall::post {
  firewall { '999 ip6 drop all':
    provider => ip6tables,
    chain    => 'INPUT',
    action   => 'drop',
    before   => undef,
  }
}
```

## Network Interface with puppet/network Module

```puppet
class profile::ipv6::interface {
  # Ubuntu/Debian: Manage /etc/network/interfaces
  network::interface { 'eth0':
    ipaddress => '10.0.0.10',
    netmask   => '255.255.255.0',
    # IPv6 managed separately via augeas/template
  }

  # Add IPv6 address via augeas
  augeas { 'eth0_ipv6':
    context => '/files/etc/network/interfaces',
    changes => [
      "set iface[. = 'eth0 inet6 static'] eth0 inet6 static",
      "set iface[. = 'eth0 inet6 static']/address 2001:db8::10",
      "set iface[. = 'eth0 inet6 static']/netmask 64",
      "set iface[. = 'eth0 inet6 static']/gateway 2001:db8::1",
    ],
  }
}
```

## Profile-Based IPv6 Deployment

```puppet
# profiles/manifests/ipv6/base.pp
class profile::ipv6::base {
  include profile::ipv6::sysctl
  include profile::ipv6::firewall
}

# profiles/manifests/ipv6/router.pp
class profile::ipv6::router {
  include profile::ipv6::base

  sysctl { 'net.ipv6.conf.all.forwarding':
    value => '1',
  }
}

# Role assignments
node 'webserver01' {
  include profile::ipv6::base
}

node 'router01' {
  include profile::ipv6::router
}
```

```bash
# Deploy to all nodes
puppet agent --test --verbose

# Check module version and update
puppet module upgrade puppetlabs-firewall

# Run tests for module
pdk test unit
```

Using Puppet Forge modules for IPv6 configuration reduces the amount of custom code needed and provides battle-tested implementations of sysctl management, firewall rules, and network configuration maintained by the Puppet community.
