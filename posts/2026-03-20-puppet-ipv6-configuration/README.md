# How to Configure IPv6 with Puppet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Puppet, IPv6, Configuration Management, Automation, Linux, Networking

Description: A guide to configuring IPv6 network settings on Linux systems using Puppet manifests, including sysctl parameters, interface configuration, and firewall rules.

Puppet enables consistent IPv6 configuration across fleets of Linux servers through declarative manifests. This guide covers the key Puppet resources and patterns for deploying IPv6 configuration at scale.

## Puppet Module Structure

```
modules/ipv6/
├── manifests/
│   ├── init.pp          # Main class
│   ├── sysctl.pp        # IPv6 kernel parameters
│   ├── firewall.pp      # ip6tables rules
│   └── interface.pp     # Network interface config
├── templates/
│   └── sysctl.conf.erb  # sysctl template
└── files/
    └── ip6tables.rules  # Firewall rules file
```

## Main IPv6 Class

```puppet
# modules/ipv6/manifests/init.pp

class ipv6 (
  Boolean $enable_forwarding = false,
  Boolean $enable_privacy    = true,
  String  $accept_ra         = '1',
  Array   $firewall_rules    = [],
) {
  include ipv6::sysctl

  if $enable_privacy {
    include ipv6::privacy
  }

  if $enable_forwarding {
    class { 'ipv6::routing':
      forwarding => true,
    }
  }
}
```

## IPv6 sysctl Configuration

```puppet
# modules/ipv6/manifests/sysctl.pp

class ipv6::sysctl {
  # Enable IPv6
  sysctl { 'net.ipv6.conf.all.disable_ipv6':
    value => '0',
  }

  sysctl { 'net.ipv6.conf.default.disable_ipv6':
    value => '0',
  }

  # Accept Router Advertisements
  sysctl { 'net.ipv6.conf.all.accept_ra':
    value => '1',
  }

  # Accept autoconfiguration (SLAAC)
  sysctl { 'net.ipv6.conf.all.autoconf':
    value => '1',
  }

  # Enable Duplicate Address Detection
  sysctl { 'net.ipv6.conf.all.dad_transmits':
    value => '1',
  }
}
```

## Using the sysctl Type

If the `herculesteam/augeasproviders_sysctl` module is not available, use augeas or exec:

```puppet
class ipv6::sysctl {
  # Using augeas for persistent sysctl settings
  augeas { 'ipv6_forwarding':
    context => '/files/etc/sysctl.conf',
    changes => [
      'set net.ipv6.conf.all.forwarding 1',
      'set net.ipv6.conf.default.forwarding 1',
    ],
    notify => Exec['reload_sysctl'],
  }

  exec { 'reload_sysctl':
    command     => '/sbin/sysctl -p',
    refreshonly => true,
  }
}
```

## IPv6 Firewall Rules with Puppet

```puppet
# modules/ipv6/manifests/firewall.pp
# Using puppetlabs/firewall module with ip6tables support

class ipv6::firewall {
  # Allow loopback
  firewall { '000 ip6 allow loopback':
    provider => ip6tables,
    chain    => 'INPUT',
    iniface  => 'lo',
    action   => 'accept',
  }

  # Allow established connections
  firewall { '001 ip6 allow established':
    provider => ip6tables,
    chain    => 'INPUT',
    state    => ['ESTABLISHED', 'RELATED'],
    action   => 'accept',
  }

  # Allow ICMPv6 (critical for IPv6 operation)
  firewall { '002 ip6 allow icmpv6':
    provider => ip6tables,
    chain    => 'INPUT',
    proto    => 'ipv6-icmp',
    action   => 'accept',
  }

  # Allow SSH over IPv6
  firewall { '010 ip6 allow ssh':
    provider => ip6tables,
    chain    => 'INPUT',
    dport    => 22,
    proto    => 'tcp',
    action   => 'accept',
  }

  # Allow HTTP and HTTPS over IPv6
  firewall { '020 ip6 allow http':
    provider => ip6tables,
    chain    => 'INPUT',
    dport    => [80, 443],
    proto    => 'tcp',
    action   => 'accept',
  }

  # Default deny
  firewall { '999 ip6 default deny':
    provider => ip6tables,
    chain    => 'INPUT',
    action   => 'drop',
  }
}
```

## IPv6 Network Interface Configuration

```puppet
# modules/ipv6/manifests/interface.pp

class ipv6::interface (
  String $interface = 'eth0',
  Optional[String] $static_address = undef,
  Optional[String] $prefix_length  = '64',
) {
  if $static_address {
    # Configure static IPv6 address
    augeas { "ipv6_static_${interface}":
      context => "/files/etc/network/interfaces",
      changes => [
        "set iface[. = '${interface} inet6 static'] ${interface} inet6 static",
        "set iface[. = '${interface} inet6 static']/address ${static_address}",
        "set iface[. = '${interface} inet6 static']/netmask ${prefix_length}",
      ],
    }
  }
}
```

## Hiera Integration for IPv6 Config

```yaml
# data/nodes/webserver01.yaml
ipv6::enable_forwarding: false
ipv6::enable_privacy: true
ipv6::accept_ra: '1'

# data/nodes/router01.yaml
ipv6::enable_forwarding: true
ipv6::enable_privacy: false
ipv6::accept_ra: '2'   # Accept RA even with forwarding enabled
```

## Applying the Module

```puppet
# site.pp or profile
node 'webserver01' {
  include ipv6
  include ipv6::firewall
}

node 'router01' {
  class { 'ipv6':
    enable_forwarding => true,
    enable_privacy    => false,
  }
}
```

```bash
# Apply configuration
puppet agent --test --verbose

# Check what would change without applying
puppet agent --test --noop
```

Puppet's declarative approach ensures IPv6 configuration is consistently applied and idempotent — running the manifest multiple times produces the same result, making it safe to apply across large server fleets.
